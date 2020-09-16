# Ansible Role: bigiq_as3_device_disaster_recovery

Performs a series of steps needed to move all AS3 application services from a unreachable BIG-IP that was a member of an Active-Standby HA pair in BIG-IQ.
This role is meant to use in the context of an RMA (Return Merchandise Authorization). 

This role is relevant when the device which needs to be replace was the target of the AS3 deployments.

## Role Variables

Available variables are listed below. For their default values, see `defaults/main.yml`.

Establishes initial connection to your BIG-IQ. These values are substituted into
your ``provider`` module parameter. These values should be the connection parameters
for the **CM BIG-IQ** device.

        provider:
          user: admin
          server: 10.1.1.4
          server_port: 443
          password: secret
          loginProviderName: tmos
          validate_certs: no

Define the variables to move all AS3 application services from the unreable BIG-IP to the remaining active BIG-IP device.
Both ``current_as3_target`` and ``new_as3_target`` must be in the same Active-Standby HA pair.

      # Working directory to store backup files on your local machine
      dir_as3: ~/tmp

      current_as3_target: 10.1.1.7 # BIG-IP device which needs the RMA
      new_as3_target: 10.1.1.8 # active BIG-IP device

## Example Playbook

    ---
    - hosts: all
      connection: local
      vars:
        provider:
          user: admin
          server: "{{ ansible_host }}"
          server_port: 443
          password: secret
          loginProviderName: tmos
          validate_certs: no

      tasks:
          - name: Move all AS3 application services from a unreachable BIG-IP that was a member of an Active-Standby HA pair in BIG-IQ
            include_role:
              name: f5devcentral.bigiq_as3_device_disaster_recovery
            vars:
              dir_as3: ~/tmp
              current_as3_target: 10.1.1.7
              new_as3_target: 10.1.1.8
            register: status

Actions to perform after the playbook is executed successfully:
- Once the AS3 application services have been re-deployed to the ``new_as3_target``, you can now remove the unreachable 
device which needs an RMA (``current_as3_target``) from BIG-IQ (remove all services first).
- Once ``current_as3_target`` is removed from BIG-IQ, you can add the new BIG-IP replacing it.
- Make sure you add it to the existing BIG-IP cluster in BIG-IQ.

## License

Apache

## Author Information

This role was created in 2020 by [Romain Jouhannet](https://github.com/rjouhann).

[1]: https://galaxy.ansible.com/f5devcentral/bigiq_pinning_deploy_objects

