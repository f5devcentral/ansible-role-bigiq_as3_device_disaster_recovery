# Ansible Role: bigiq_as3_device_disaster_recovery

Performs a series of steps needed in BIG-IQ to replace a RMA (Return Merchandise Authorization) device which had AS3 application services deployed.

The role is supporting RMA device that was a member of an **Active-Standby HA pair** or **Standalone** in BIG-IQ. 
The role is relevant when the RMA device was the **target of AS3 deployments**.

Steps executed by the role automatically:
1. Backup AS3 declarations from RMA device
2. Delete the AS3 application services on BIG-IQ dashboard (apps won't be deleted on the BIG-IP but only on BIG-IQ)
3. Remove the RMA device from BIG-IQ
4. Add back the RMA device in BIG-IQ (assuming it has been replaced and UCS backup restored using the same IP address as before)
5. Re-deploy the AS3 application services on the new target device (no service impact)

# Prerequisites

- The RMA device have been replaced before running this role and is up and running, HA cluster sync (*if not standalone*)
- Install following galaxy roles:
  - ``ansible-galaxy install f5devcentral.atc_deploy --force``
  - ``ansible-galaxy install f5devcentral.bigiq_move_app_dashboard --force``
- Install following galaxy collection:
  - ``ansible-galaxy collection install f5networks.f5_modules --force`` (for latest up to date [bigiq_device_discovery](https://docs.ansible.com/ansible/latest/modules/bigiq_device_discovery_module.html) module)

# Limitations

- RMA device IP address must be the same after the device has been replaced & restore
- This role does NOT save the Application Services **Custom Application Roles** assigned to a user or groups of users for the applications hosted on the RMA device. 
You may not use this role if you are in this case. If you are interested to support the user/application roles relation [open an issue on GitHub](https://github.com/f5devcentral/ansible-role-bigiq_as3_device_disaster_recovery/issues).

# Notes

- The Analytics history on BIG-IQ for this device won't be lost.
- The re-discover & re-import of the RMA device repaired will use the following conflict resolution policy **Use BIG-IP** by default.
- If you had users assigned to the AS3 application services in RMA device, you will need to re-assign the application services roles to those users after the role is executed

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
          auth_provider: tmos
          validate_certs: false

- RMA device is part of a BIG-IP **Active-Standby HA pair**: ``current_as3_target`` and ``new_as3_target`` must be part of the same Cluster in BIG-IQ.
- RMA device is a **Standalone** BIG-IP: ``current_as3_target`` and ``new_as3_target`` can be the same or different IP.

      # Working directory to store backup files on your local machine
      dir_as3: ~/tmp

      current_as3_target: 10.1.1.7 # RMA device
      new_as3_target: 10.1.1.8 # new AS3 target
      device_username: admin # RMA device user
      device_password: secret # RMA device password

      standalone: false # Set to true if RMA device is a standalone device

      # Conflict resolution options
      conflict_policy: use_bigip
      device_conflict_policy: use_bigip
      versioned_conflict_policy: keep_version

      # Cluster Sync
      use_bigiq_sync: yes

      # Stats options  
      enable: yes 
      stat_modules: 
        - device
        - ltm
        - dns
        - afm
      interval: 60
      zone: default

      # Access only
      access_group_name: myaccessgroup
      access_conflict_policy: use_bigip
      access_group_first_device: yes

See [bigiq_device_discovery](https://docs.ansible.com/ansible/latest/modules/bigiq_device_discovery_module.html) for Conflict resolution and stats options.

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
          auth_provider: tmos
          validate_certs: false

      tasks:
          - name: Move all AS3 application services from BIG-IP device which needed a RMA
            include_role:
              name: f5devcentral.bigiq_as3_device_disaster_recovery
            vars:
              dir_as3: ~/tmp
              current_as3_target: 10.1.1.7
              new_as3_target: 10.1.1.8
              device_username: admin
              device_password: secret
            register: status

## Troubleshooting

Backups of AS3 declarations are created in ``dir_as3`` folder specified in your playbook before the application services are removed from BIG-IQ.

You may use this in the event the role failed to restore the existing AS3 application services on BIG-IQ.

You will find the backup json files which can be used to restore the AS3 configuration 
using the [F5 automation tool chain (ATC) deploy declaration](https://galaxy.ansible.com/f5devcentral/atc_deploy) galaxy role.

Please, make sure you verify the content of the json files before restoring any AS3 declarations on BIG-IQ.

    ---
    - hosts: all
      connection: local
      vars:
        provider:
          user: admin
          server: "{{ ansible_host }}"
          server_port: 443
          password: secret
          auth_provider: tmos
          validate_certs: false

      tasks:
          - name: ATC POST
            include_role:
              name: f5devcentral.atc_deploy
            vars:
              atc_service: AS3
              atc_method: POST
              atc_declaration_file: "~/tmp/10.1.1.7_bigip.json.bkp"
              atc_delay: 15
              atc_retries: 30
            register: atc_AS3_status

          - name: Move AS3 application service(s) in BIG-IQ application dashboard.
            include_role:
              name: f5devcentral.bigiq_move_app_dashboard
            vars:
                apps: "{{ lookup('file', '~/tmp/10.1.1.4_bigiq_apps_mapping.json.bkp') }}"
            register: status

``10.1.1.7`` is the BIG-IP. ``10.1.1.4`` is the BIG-IQ (or ``ansible_host``).

## License

Apache

## Author Information

This role was created in 2020 by [Romain Jouhannet](https://github.com/rjouhann).

[1]: https://galaxy.ansible.com/f5devcentral/bigiq_pinning_deploy_objects

