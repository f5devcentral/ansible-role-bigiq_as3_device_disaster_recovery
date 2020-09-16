# Ansible Role: bigiq_as3_device_disaster_recovery

Performs a series of steps needed to move all AS3 Application Service(s) from a BIG-IP device 
which needs an RMA (Return Merchandise Authorization) to the active device serving the traffic.

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

Define the variables to move all AS3 application services from to the active BIG-IP device.

      # Working directory to store backup files on your local machine
      dir_as3: ~/tmp

      current_as3_target: 10.1.1.7 # BIG-IP device which needs the RMA
      new_as3_target: 10.1.1.8 # active BIG-IP device

      # Name of the Application in BIG-IQ Dashboard which will contain the App Services after moving them to the new_as3_target
      new_bigiq_app_name: "App Services moved to new target"

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
          - name: Move AS3 application service(s) in BIG-IQ application dashboard.
            include_role:
              name: f5devcentral.bigiq_as3_device_disaster_recovery
            vars:
              dir_as3: ~/tmp
              current_as3_target: 10.1.1.7
              new_as3_target: 10.1.1.8
              new_bigiq_app_name: "App Services moved to new target"
            register: status

Action after the playbook is executed successfully:
- Once the AS3 application services have been re-deployed to the ``new_as3_target``, you can now remove the unreachable 
device which needs an RMA (``current_as3_target``) from BIG-IQ (remove all services first).
- Once ``current_as3_target`` is removed from BIG-IQ, you can add the new BIG-IP replacing it.
- Make sure you add it to the existing BIG-IP cluster in BIG-IQ.

## License

Apache

## Author Information

This role was created in 2020 by [Romain Jouhannet](https://github.com/rjouhann).

[1]: https://galaxy.ansible.com/f5devcentral/bigiq_pinning_deploy_objects

