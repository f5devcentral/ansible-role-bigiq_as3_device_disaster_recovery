# Ansible Role: bigiq_as3_device_disaster_recovery

Performs a series of steps needed to move all AS3 application services from BIG-IP device which needed a RMA (Return Merchandise Authorization).

The role is supporting RMA device that was a member of an **Active-Standby HA pair** or **Standalone** in BIG-IQ. 
The role is relevant when the RMA device was the **target of AS3 deployments**.

Steps executed by the role:
1. Backup AS3 declarations from RMA device
2. Delete the AS3 application services on BIG-IQ dashboard (apps won't be deleted on the BIG-IP but only on BIG-IQ)
3. Re-deploy the AS3 application services on the new target device (no service impact)

**CAUTION**
This role does NOT save the Application Services **Custom Application Roles** assigned to a user or groups of users for the applications hosted on the RMA device. You may not use this role if you are in this case. If you are interested to support the user/application roles relation [open an issue on GitHub](https://github.com/f5devcentral/ansible-role-bigiq_as3_device_disaster_recovery/issues).

**Actions to perform after the role is used:**
- Once the AS3 application services have been re-deployed to the ``new_as3_target``, you can now remove the RMA device (``current_as3_target``) from BIG-IQ (remove all services first).
- Once ``current_as3_target`` is removed from BIG-IQ, you can add it back after the device has been replaced and UCS backup restored.
- Make sure you add it to the existing BIG-IP cluster in BIG-IQ if this device was part of a Active-Standby HA pair.
- Make sure sure you Re-Discover & Re-import ALL the modules for ``current_as3_target`` and `new_as3_target`` once RMA device replaced.
  (Conflict Resolution Policy: Use BIG-IP for Device Object Conflict)
- If you had users assigned to the AS3 application services in in ``current_as3_target``, you will need to re-assign the application services roles to those users.

Note the Analytics history on BIG-IQ for this device won't be lost.

# Prerequisites

- Install following galaxy roles before using this role:
  - ansible-galaxy install f5devcentral.atc_deploy
  - ansible-galaxy install f5devcentral.bigiq_move_app_dashboard

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

- RMA device is part of a BIG-IP **Active-Standby HA pair**: ``current_as3_target`` and ``new_as3_target`` must be part of the same Cluster in BIG-IQ.
- RMA device is a **Standalone** BIG-IP: ``current_as3_target`` and ``new_as3_target`` can be the same or different IP.

      # Working directory to store backup files on your local machine
      dir_as3: ~/tmp

      current_as3_target: 10.1.1.7 # RMA device
      new_as3_target: 10.1.1.8 # new AS3 target

      standalone: false # Set to true if RMA device is a standalone device

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
          - name: Move all AS3 application services from BIG-IP device which needed a RMA
            include_role:
              name: f5devcentral.bigiq_as3_device_disaster_recovery
            vars:
              dir_as3: ~/tmp
              current_as3_target: 10.1.1.7
              new_as3_target: 10.1.1.8
            register: status

## License

Apache

## Author Information

This role was created in 2020 by [Romain Jouhannet](https://github.com/rjouhann).

[1]: https://galaxy.ansible.com/f5devcentral/bigiq_pinning_deploy_objects

