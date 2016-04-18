[![Circle CI](https://circleci.com/gh/daniel-rhoades/aws-elb-role.svg?style=svg&circle-token=13089bfa0d6f6680112eaf4cda24a949b560ecef)](https://circleci.com/gh/daniel-rhoades/aws-elb-role)

aws-elb-role
============

Ansible role for simplifying the provisioning and decommissioning an ELB within an AWS account.

For more detailed on information on the creating:

* Auto-scaling Groups: http://docs.ansible.com/ansible/ec2_elb_lb__module.html;
* Route 53: http://docs.ansible.com/ansible/route53_module.html.

This role will completely setup a multi-AZ ELB and register with a Route 53 record in a zone of your choice.

Requirements
------------

Requires the latest Ansible EC2 support modules along with Boto.

You will also need to configure your Ansible environment for use with AWS, see http://docs.ansible.com/ansible/guide_aws.html.

Role Variables
--------------

Defaults:

* elb_purge_subnets: Purge existing listeners on ELB that are not found in listeners, defaults to true;
* elb_cross_az_load_balancing: Distribute load across all configured Availability Zones, defaults to true;
* elb_connection_draining_timeout: Wait a specified timeout allowing connections to drain before terminating an instance, defaults to 60 seconds;
* elb_listeners: Listeners for the ELB, 80:80 is available by default;
* elb_health_check: Health check to run to determine instance health, default is to make a TCP connection to port 80;
* route53_overwrite: Overwrites existing entries if required, defaults to true;
* route53_alias_evaluate_target_health: If you want Route 53 to monitor target health, defaults to true;
* ec2_elb_state: State of the ELB, defaults to "present";
* route53_zone_state: State of the Route 53 zone, defaults to "present".

Required variables:

* elb_name: You must specify the name of the ELB you wish to create, e.g. my-elb;
* elb_security_groups: You must specify the security groups to be applied to the ELB you wish to create, see the example playbook below;
* elb_region: You must specify the region you wish to create the ELB within, e.g. eu-west-1;
* elb_subnets: You must specify the subnets you wish to make the ELB available within, see the example playbook below;
* route53_zone: You must specify the name of the zone you wish to define, e.g. example.com;
* route53_host: You must specify the name of the host you wish to define on the zone, e.g. www.example.com.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Before using this role you will need to install the role, the simplist way to do this is: `ansible-galaxy install daniel-rhoades.aws-elb-role`.

For completeness the examples create

* VPC to hold the ECS cluster, using the role: `daniel-rhoades.aws-vpc`;
* EC2 Security Groups to apply to the EC2 instances, using the role: `daniel-rhoades.aws-security-group`.

After creation you can register your EC2 instances to the ELB.  The example expects `my_route53_zone` to be passed in as a command line environment variable.

```
- name: My System | Provision all required infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    my_vpc_name: "my_example_vpc"
    my_vpc_region: "eu-west-1"
    my_vpc_cidr: "172.40.0.0/16"
    everywhere_cidr: "0.0.0.0/0"

    # Subnets within the VPC
    my_vpc_subnets:
      - cidr: "172.40.10.0/24"
        az: "{{ my_vpc_region }}a"

      - cidr: "172.40.20.0/24"
        az: "{{ my_vpc_region }}b"

    # Allow the subnets to route to the outside world
    my_public_subnet_routes:
      - subnets:
          - "{{ my_vpc_subnets[0].cidr }}"
          - "{{ my_vpc_subnets[1].cidr }}"
        routes:
          - dest: "{{ everywhere_cidr }}"
            gw: igw

    # Inbound security groups, e.g. public facing services like a load balancer
    my_inbound_security_groups:
      - sg_name: inbound-web
        sg_description: allow http and https access (public)
        sg_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: "{{ everywhere_cidr }}"

          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: "{{ everywhere_cidr }}"

      # Only allow SSH access from within the VPC, to access any services within the VPC you would need to create a
      # temporary bastion host
      - sg_name: inbound-ssh
        sg_description: allow ssh access
        sg_rules:
         - proto: tcp
           from_port: 22
           to_port: 22
           cidr_ip: "{{ my_vpc_cidr }}"

    # Internal inbound security groups, e.g. services which should not be directly accessed outside the VPC, such as
    # the web servers behind the load balancer.
    #
    # This has to be a file as it needs to be dynamically included after the inbound security groups have been created
    my_internal_inbound_security_groups_file: "internal-securitygroups.yml"

    # Outbound rules, e.g. what services can the web servers access by themselves
    my_outbound_security_groups:
      - sg_name: outbound-all
        sg_description: allows outbound traffic to any IP address
        sg_rules:
          - proto: all
            cidr_ip: "{{ everywhere_cidr }}"

    # Name of the ELB
    my_elb_name: "my-elb"

    # Name of the host to create on the Route 53 zone
    my_route53_host: "my-service-test"

  roles:
    # Provision networking
    - {
        role: daniel-rhoades.aws-vpc,
        vpc_name: "{{ my_vpc_name }}",
        vpc_region: "{{ my_vpc_region }}",
        vpc_cidr_block: "{{ my_vpc_cidr }}",
        vpc_subnets: "{{ my_vpc_subnets }}",
        public_subnet_routes: "{{ my_public_subnet_routes }}"
      }

    # Provision security groups
    - {
        role: daniel-rhoades.aws-security-groups,
        vpc_region: "{{ my_vpc_region }}",
        vpc_id: "{{ vpc.vpc_id }}",
        ec2_group_inbound_sg: "{{ my_inbound_security_groups }}",
        ec2_group_internal_inbound_sg_file: "{{ my_internal_inbound_security_groups_file }}",
        ec2_group_outbound_sg: "{{ my_outbound_security_groups }}"
      }

    # Provision an ELB and register it with an alias in Route 53
    - {
        role: aws-elb-role,
        elb_name: "{{ my_elb_name }}",
        vpc_name: "{{ my_vpc_name }}",
        elb_security_groups: [
          "{{ ec2_group_inbound_sg.results[0].group_id }}"
          ],
        elb_region: "{{ my_vpc_region }}",
        elb_subnets: [
           "{{ vpc.subnets[0].id }}",
           "{{ vpc.subnets[1].id }}"
         ],
        route53_zone: "{{ my_route53_zone }}",
        route53_host: "{{ my_route53_host }}"
      }
```

The example `internal-securitygroups.yml` looks like:

```
ec2_group_internal_inbound_sg:
  - sg_name: inbound-web-internal
    sg_description: allow http and https access (from load balancer only)
    sg_rules:
      - proto: tcp
        from_port: 80
        to_port: 80
        group_id: "{{ ec2_group_inbound_sg.results[0].group_id }}"
```

To decommission the ELB:

```
- name: My System | Decommission all required infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    my_vpc_name: "my_example_vpc"
    my_vpc_region: "eu-west-1"
    my_vpc_cidr: "172.40.0.0/16"
    everywhere_cidr: "0.0.0.0/0"

    # Subnets within the VPC
    my_vpc_subnets:
      - cidr: "172.40.10.0/24"
        az: "{{ my_vpc_region }}a"

      - cidr: "172.40.20.0/24"
        az: "{{ my_vpc_region }}b"

    # Allow the subnets to route to the outside world
    my_public_subnet_routes:
      - subnets:
          - "{{ my_vpc_subnets[0].cidr }}"
          - "{{ my_vpc_subnets[1].cidr }}"
        routes:
          - dest: "{{ everywhere_cidr }}"
            gw: igw

    # Name of the ECS cluster to create
    my_ecs_cluster_name: "my-cluster"

    # Name of the ELB
    my_elb_name: "my-elb"

    # Name of the host to create on the Route 53 zone
    my_route53_host: "my-service-test"

  roles:
    # Provision networking
    - {
        role: daniel-rhoades.aws-vpc,
        vpc_name: "{{ my_vpc_name }}",
        vpc_region: "{{ my_vpc_region }}",
        vpc_cidr_block: "{{ my_vpc_cidr }}",
        vpc_subnets: "{{ my_vpc_subnets }}",
        public_subnet_routes: "{{ my_public_subnet_routes }}"
      }

    # Provision security groups
    - {
        role: daniel-rhoades.aws-security-groups,
        vpc_region: "{{ my_vpc_region }}",
        vpc_id: "{{ vpc.vpc_id }}",
        ec2_group_inbound_sg: "{{ my_inbound_security_groups }}",
        ec2_group_internal_inbound_sg_file: "{{ my_internal_inbound_security_groups_file }}",
        ec2_group_outbound_sg: "{{ my_outbound_security_groups }}"
      }

    # Decomission the ELB (REST endpoint)
    - {
        role: aws-elb-role,
        ec2_elb_state: "absent",
        vpc_name: "{{ my_vpc_name }}",
        elb_name: "{{ my_elb_name }}",
        route53_zone: "{{ my_route53_zone }}",
        route53_host: "{{ my_route53_host }}"
      }
```

License
-------

MIT

Author Information
------------------

Daniel Rhoades (https://github.com/daniel-rhoades)
