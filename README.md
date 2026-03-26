---
- name: Upgrade Splunk UF to 10.2.0 and manage with systemd
  hosts: all
  become: true
  gather_facts: false
  serial: 25

  vars:
    target_version: "10.2.0"
    splunk_home: "/opt/splunkforwarder"
    splunk_bin: "{{ splunk_home }}/bin/splunk"
    splunk_unit: "SplunkForwarder.service"
    splunk_unit_path: "/etc/systemd/system/SplunkForwarder.service"

    splunk_pkg_url: "http://repo.abbvienet.com/linux/splunkforwarder"
    splunk_x86_rpm_url: >-
      {{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65.x86_64.rpm
    splunk_arm_rpm_url: >-
      {{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65.aarch64.rpm
    splunk_amd_deb_url: >-
      {{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65-linux-amd64.deb
    splunk_arm_deb_url: >-
      {{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65-linux-arm64.deb

    x86_rpm_dest: "/tmp/splunkforwarder-{{ target_version }}.x86_64.rpm"
    arm_rpm_dest: "/tmp/splunkforwarder-{{ target_version }}.aarch64.rpm"
    amd_deb_dest: "/tmp/splunkforwarder-{{ target_version }}.amd64.deb"
    arm_deb_dest: "/tmp/splunkforwarder-{{ target_version }}.arm64.deb"

  pre_tasks:
    - name: Connectivity check
      ansible.builtin.command: uptime
      register: uptime_check
      changed_when: false
      ignore_unreachable: true
      ignore_errors: true

    - name: Skip unreachable hosts
      ansible.builtin.meta: end_host
      when: uptime_check is failed

    - name: Gather minimal facts
      ansible.builtin.setup:
        gather_subset:
          - "!all"
          - "distribution"
          - "hardware"

    - name: Skip unsupported hosts
      ansible.builtin.meta: end_host
      when:
        - ansible_os_family not in ["RedHat", "Debian"] or
          ansible_architecture not in ["x86_64", "amd64", "arm64", "aarch64"]

  tasks:
    - name: Check current version
      ansible.builtin.shell: >-
        {{ splunk_bin }} version 2>&1 |
        grep -Eo "[0-9]+\.[0-9]+\.[0-9]+" |
        head -n 1
      args:
        executable: /bin/bash
      register: current_out
      changed_when: false
      failed_when: false

    - name: Set version facts
      ansible.builtin.set_fact:
        current_version: >-
          {{ current_out.stdout | default('not_installed') | trim
             | default('not_installed') }}
        upgrade_needed: >-
          {{ (current_out.stdout | default('') | trim) != target_version }}

    - name: Stop Splunk before upgrade
      ansible.builtin.command: "{{ splunk_bin }} stop"
      register: splunk_stop_result
      changed_when: false
      failed_when: false
      when:
        - upgrade_needed
        - current_version != "not_installed"

    - name: Download x86 RPM
      ansible.builtin.get_url:
        url: "{{ splunk_x86_rpm_url }}"
        dest: "{{ x86_rpm_dest }}"
        mode: "0644"
      when:
        - upgrade_needed
        - ansible_os_family == "RedHat"
        - ansible_architecture in ["x86_64", "amd64"]

    - name: Download arm RPM
      ansible.builtin.get_url:
        url: "{{ splunk_arm_rpm_url }}"
        dest: "{{ arm_rpm_dest }}"
        mode: "0644"
      when:
        - upgrade_needed
        - ansible_os_family == "RedHat"
        - ansible_architecture in ["arm64", "aarch64"]

    - name: Install x86 RPM
      ansible.builtin.dnf:
        name: "{{ x86_rpm_dest }}"
        state: present
        disable_gpg_check: true
      when:
        - upgrade_needed
        - ansible_os_family == "RedHat"
        - ansible_architecture in ["x86_64", "amd64"]

    - name: Install arm RPM
      ansible.builtin.dnf:
        name: "{{ arm_rpm_dest }}"
        state: present
        disable_gpg_check: true
      when:
        - upgrade_needed
        - ansible_os_family == "RedHat"
        - ansible_architecture in ["arm64", "aarch64"]

    - name: Download amd64 DEB
      ansible.builtin.get_url:
        url: "{{ splunk_amd_deb_url }}"
        dest: "{{ amd_deb_dest }}"
        mode: "0644"
      when:
        - upgrade_needed
        - ansible_os_family == "Debian"
        - ansible_architecture in ["x86_64", "amd64"]

    - name: Download arm64 DEB
      ansible.builtin.get_url:
        url: "{{ splunk_arm_deb_url }}"
        dest: "{{ arm_deb_dest }}"
        mode: "0644"
      when:
        - upgrade_needed
        - ansible_os_family == "Debian"
        - ansible_architecture in ["arm64", "aarch64"]

    - name: Install amd64 DEB
      ansible.builtin.apt:
        deb: "{{ amd_deb_dest }}"
        state: present
      when:
        - upgrade_needed
        - ansible_os_family == "Debian"
        - ansible_architecture in ["x86_64", "amd64"]

    - name: Install arm64 DEB
      ansible.builtin.apt:
        deb: "{{ arm_deb_dest }}"
        state: present
      when:
        - upgrade_needed
        - ansible_os_family == "Debian"
        - ansible_architecture in ["arm64", "aarch64"]

    - name: Disable existing boot-start
      ansible.builtin.command: "{{ splunk_bin }} disable boot-start"
      register: disable_boot_start_result
      changed_when: false
      failed_when: false

    - name: Remove old SplunkForwarder unit
      ansible.builtin.file:
        path: "{{ splunk_unit_path }}"
        state: absent

    - name: Remove old splunk unit
      ansible.builtin.file:
        path: "/etc/systemd/system/splunk.service"
        state: absent

    - name: Enable boot-start with systemd
      ansible.builtin.command: >-
        {{ splunk_bin }} enable boot-start
        --accept-license
        --answer-yes
        --no-prompt
        -systemd-managed 1
        -user splunk
      register: enable_boot_start_result
      changed_when: false
      failed_when: false

    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Check SplunkForwarder.service exists
      ansible.builtin.stat:
        path: "{{ splunk_unit_path }}"
      register: unit_file

    - name: Fail if systemd service file was not created
      ansible.builtin.fail:
        msg: "SplunkForwarder.service was not created"
      when: not unit_file.stat.exists

    - name: Enable and restart SplunkForwarder.service
      ansible.builtin.systemd:
        name: "{{ splunk_unit }}"
        enabled: true
        state: restarted
        daemon_reload: true

    - name: Verify version
      ansible.builtin.shell: >-
        {{ splunk_bin }} version 2>&1 |
        grep -Eo "[0-9]+\.[0-9]+\.[0-9]+" |
        head -n 1
      args:
        executable: /bin/bash
      register: verify_out
      changed_when: false
      failed_when: false

    - name: Verify systemd service is active
      ansible.builtin.command: "systemctl is-active {{ splunk_unit }}"
      register: svc_active
      changed_when: false
      failed_when: false

    - name: Verify systemd service is enabled
      ansible.builtin.command: "systemctl is-enabled {{ splunk_unit }}"
      register: svc_enabled
      changed_when: false
      failed_when: false

    - name: Fail if validation failed
      ansible.builtin.fail:
        msg: >-
          Validation failed:
          version={{ verify_out.stdout | default('') | trim }},
          active={{ svc_active.stdout | default('') | trim }},
          enabled={{ svc_enabled.stdout | default('') | trim }}
      when:
        - (verify_out.stdout | default('') | trim) != target_version or
          (svc_active.stdout | default('') | trim) != 'active' or
          (svc_enabled.stdout | default('') | trim) != 'enabled'

    - name: Final status
      ansible.builtin.debug:
        msg: >-
          Host={{ inventory_hostname }}
          Version={{ verify_out.stdout | default('') | trim }}
          Active={{ svc_active.stdout | default('') | trim }}
          Enabled={{ svc_enabled.stdout | default('') | trim }}
