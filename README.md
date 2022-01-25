# Ansible Role: bigiq_as3_device_disaster_recovery

Performs a series of steps needed in BIG-IQ to remove and add again a BIG-IP managed by BIG-IQ with AS3 or Legacy application services deployed on it.

The role is relevant in the following cases:
  - You have done an RMA (Return Merchandise Authorization) on a device managed by BIG-IQ with application services deployed on it which prevents you to replace the device.
  - F5 Support recommends to remove/re-add BIG-IP device from BIG-IQ to resolve potential problems but this device has AS3 or Legacy application services deployed on it.

Steps executed by the ansible galaxy role automatically: *(role will pause between each step)*
1. Backup AS3 declarations and Legacy App Services from the device specified
2. Delete the Application Services in BIG-IQ dashboard belonging to the device specified (app services won't be deleted on the BIG-IP but only on BIG-IQ from the Application Tab)
3. Remove the device specified from BIG-IQ
4. Re-discover and re-import the device specified in BIG-IQ (and re-discover other device part of the HA cluster, if applicable)
5. Re-deploy the AS3 and Legacy application services on the specified device (no service impact) (using other device part of the HA cluster, if applicable)

# Prerequisites

- Make sure device part of the same cluster can be successfully discovered and imported in BIG-IQ
- In the case of RMA, make sure the device has been replaced before running this role and is up and running, HA cluster sync
- Install the following galaxy roles:
  - ``ansible-galaxy install f5devcentral.atc_deploy --force``
  - ``ansible-galaxy install f5devcentral.bigiq_move_app_dashboard --force``
- Install the following galaxy collection:
  - ``ansible-galaxy collection install f5networks.f5_modules --force`` (see [bigiq_device_discovery](https://docs.ansible.com/ansible/latest/modules/bigiq_device_discovery_module.html) module)

# Limitations

- The role will only restore **AS3 application services** and **Legacy Application Services** (no App Services deployed using Service Catalog templates).
For app services deployed using Service Catalog templates, it is recommended to [convert](https://clouddocs.f5.com/training/community/big-iq-cloud-edition/html/class1/module6/lab4.html) it to Legacy App Services.
- In the case of RMA, the device IP address must be the same after the device has been replaced & restored.
- This role does NOT save the **Custom Application Roles** assigned to a user or groups of users for the applications hosted on the device. 
You may not use this role if you are in this case. If you are interested to support the user/application roles relation [open an issue on GitHub](https://github.com/f5devcentral/ansible-role-bigiq_as3_device_disaster_recovery/issues).

# Notes

- In case you have an HA cluster, the application services will be re-deploy on the other BIG-IP pair ``bigip2_target`` (step 5).
- The Analytics history on BIG-IQ for this device won't be lost but BIG-IQ won't collect analytics when the device is removed then re-added to the BIG-IQ.
- The re-discover & re-import of the device specified will use the following conflict resolution policy **Use BIG-IP** by default.
- If you had users assigned to the AS3 or Legacy application services in the device, you will need to re-assign the application services roles to those users after the role is executed

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

Device is part of a BIG-IP **Active-Standby HA pair**: ``bigip1_target`` and ``bigip2_target`` must be part of the same HA cluster in BIG-IQ. 
If the device is not part of a HA cluster, set ``standalone`` to ``true`` and do not include ``bigip2_target``.

      # Working directory to store backup files on your local machine
      dir_as3: ~/tmp

      bigip1_target: 10.1.1.7 # BIG-IP device
      bigip2_target: 10.1.1.8 # BIG-IP device part of the same HA cluster
      device_username: admin # BIG-IP device user
      device_password: secret # BIG-IP device password
      device_port: 443  # BIG-IP device port

      standalone: false # Set to true if device is standalone or part of an HA cluster

      # Skip step 3 and 4 (add and remove BIG-IP device)
      skip_add_remove_rma_device: false

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
      access_group_name: myAccessGroup
      access_conflict_policy: use_bigip
      access_group_first_device: yes

See [bigiq_device_discovery](https://docs.ansible.com/ansible/latest/modules/bigiq_device_discovery_module.html) for Conflict resolution and stats options.

## Example Playbook - HA cluster use case

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
          - name: BIG-IP Device disaster recovery - HA use case
            include_role:
              name: f5devcentral.bigiq_as3_device_disaster_recovery
            vars:
              dir_as3: ~/tmp
              bigip1_target: 10.1.1.7
              bigip2_target: 10.1.1.8
              device_username: admin
              device_password: secret
            register: status

## Example Playbook - Standalone use case

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
          - name: BIG-IP Device disaster recovery - Standalone use case
            include_role:
              name: f5devcentral.bigiq_as3_device_disaster_recovery
            vars:
              dir_as3: ~/tmp
              bigip1_target: 10.1.1.7
              standalone: true
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
          - name: Restore AS3 declaration backup from 10.1.1.7
            include_role:
              name: f5devcentral.atc_deploy
            vars:
              atc_service: AS3
              atc_method: POST
              atc_declaration_file: "~/tmp/10.1.1.7_bigip.json.bkp"
              atc_delay: 15
              atc_retries: 30
            register: atc_AS3_status

          - name: Move Restored AS3 application service(s) in original BIG-IQ application dashboard.
            include_role:
              name: f5devcentral.bigiq_move_app_dashboard
            vars:
                apps: "{{ lookup('file', '~/tmp/10.1.1.4_bigiq_apps_mapping.json.bkp') }}"
            register: status

``10.1.1.7`` is the BIG-IP. ``10.1.1.4`` and ``ansible_host`` is the BIG-IQ.

## License

Apache

## Author Information

This role was created in 2021 by [Romain Jouhannet](https://github.com/rjouhann).

[1]: https://galaxy.ansible.com/f5devcentral/bigiq_pinning_deploy_objects

