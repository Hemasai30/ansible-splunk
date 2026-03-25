---
- name: Upgrade Splunk UF to 10.2.0 (AWS-friendly, dynamic systemd support)
  hosts: all
  become: yes
  gather_facts: no
  serial: 25

  vars:
    target_version: "10.2.0"
    splunk_home: "/opt/splunkforwarder"
    splunk_bin: "{{ splunk_home }}/bin/splunk"

    splunk_pkg_url: "http://repo.abbvienet.com/linux/splunkforwarder"
    splunk_x86_rpm_url: "{{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65.x86_64.rpm"
    splunk_arm_rpm_url: "{{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65.aarch64.rpm"
    splunk_amd_deb_url: "{{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65-linux-amd64.deb"
    splunk_arm_deb_url: "{{ splunk_pkg_url }}/splunkforwarder-10.2.0-d749cb17ea65-linux-arm64.deb"

    x86_rpm_dest: "/tmp/splunkforwarder-{{ target_version }}.x86_64.rpm"
    arm_rpm_dest: "/tmp/splunkforwarder-{{ target_version }}.aarch64.rpm"
    amd_deb_dest: "/tmp/splunkforwarder-{{ target_version }}.amd64.deb"
    arm_deb_dest: "/tmp/splunkforwarder-{{ target_version }}.arm64.deb"

  pre_tasks:
    - name: Connectivity check
      command: uptime
      register: uptime_check
      changed_when: false
      ignore_unreachable: true
      ignore_errors: true

    - name: Skip unreachable hosts
      meta: end_host
      when: uptime_check is failed

    - name: Gather minimal facts
      setup:
        gather_subset:
          - "!all"
          - "distribution"
          - "hardware"

    - name: Skip unsupported architectures
      meta: end_host
      when: ansible_architecture not in ["x86_64", "amd64", "arm64", "aarch64"]

    - name: Initialize host status
      set_fact:
        reached_host: true
        host_failed: false
        failure_reason: ""
        current_version: "not_installed"
        version_verified: ""
        upgrade_needed: true
        splunk_running: false
        upgrade_success: false
        systemd_unit_exists: false
        systemd_unit_name: ""
        systemd_unit_path: ""

  tasks:
    - name: Check splunk binary exists
      stat:
        path: "{{ splunk_bin }}"
      register: splunk_bin_stat

    - name: Get current version if installed
      shell: "{{ splunk_bin }} version 2>&1 | grep -Eo '[0-9]+\.[0-9]+\.[0-9]+' | head -n 1"
      register: current_out
      changed_when: false
      failed_when: false
      when: splunk_bin_stat.stat.exists

    - name: Set current version facts
      set_fact:
        current_version: "{{ current_out.stdout | default('not_installed') | trim | default('not_installed') }}"
        upgrade_needed: "{{ (current_out.stdout | default('') | trim) != target_version }}"

    - name: Show host status
      debug:
        msg: "Host={{ inventory_hostname }} Current={{ current_version }} Target={{ target_version }} UpgradeNeeded={{ upgrade_needed }}"

    - name: Upgrade workflow
      block:
        - name: Stop splunk if installed and upgrade needed
          shell: "timeout 120 {{ splunk_bin }} stop"
          register: stop_result
          changed_when: false
          failed_when: false
          when: upgrade_needed and current_version != 'not_installed'

        - name: Download x86 RPM
          get_url:
            url: "{{ splunk_x86_rpm_url }}"
            dest: "{{ x86_rpm_dest }}"
            mode: "0644"
            timeout: 60
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["x86_64", "amd64"]

        - name: Download arm RPM
          get_url:
            url: "{{ splunk_arm_rpm_url }}"
            dest: "{{ arm_rpm_dest }}"
            mode: "0644"
            timeout: 60
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["arm64", "aarch64"]

        - name: Install x86 RPM
          shell: "rpm -Uvh --force --nodeps {{ x86_rpm_dest }}"
          register: rpm_install_x86
          changed_when: true
          failed_when: false
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["x86_64", "amd64"]

        - name: Install arm RPM
          shell: "rpm -Uvh --force --nodeps {{ arm_rpm_dest }}"
          register: rpm_install_arm
          changed_when: true
          failed_when: false
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["arm64", "aarch64"]

        - name: Download amd64 DEB
          get_url:
            url: "{{ splunk_amd_deb_url }}"
            dest: "{{ amd_deb_dest }}"
            mode: "0644"
            timeout: 60
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["x86_64", "amd64"]

        - name: Download arm64 DEB
          get_url:
            url: "{{ splunk_arm_deb_url }}"
            dest: "{{ arm_deb_dest }}"
            mode: "0644"
            timeout: 60
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["arm64", "aarch64"]

        - name: Install amd64 DEB
          apt:
            deb: "{{ amd_deb_dest }}"
            state: present
          register: deb_install_amd
          failed_when: false
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["x86_64", "amd64"]

        - name: Install arm64 DEB
          apt:
            deb: "{{ arm_deb_dest }}"
            state: present
          register: deb_install_arm
          failed_when: false
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["arm64", "aarch64"]

        - name: Enable Splunk boot-start
          shell: "{{ splunk_bin }} enable boot-start --accept-license --answer-yes --no-prompt"
          register: boot_start_result
          changed_when: false
          failed_when: false
          when: upgrade_needed

        - name: Check for SplunkForwarder.service
          stat:
            path: "/etc/systemd/system/SplunkForwarder.service"
          register: splunkforwarder_unit_stat

        - name: Check for splunk.service
          stat:
            path: "/etc/systemd/system/splunk.service"
          register: splunk_unit_stat

        - name: Check vendor SplunkForwarder.service
          stat:
            path: "/usr/lib/systemd/system/SplunkForwarder.service"
          register: splunkforwarder_vendor_unit_stat

        - name: Check vendor splunk.service
          stat:
            path: "/usr/lib/systemd/system/splunk.service"
          register: splunk_vendor_unit_stat

        - name: Create Splunk boot-start service file if missing
          shell: "{{ splunk_bin }} enable boot-start --accept-license --answer-yes --no-prompt"
          register: boot_start_create_result
          changed_when: false
          failed_when: false
          when: >
            upgrade_needed and
            not splunkforwarder_unit_stat.stat.exists and
            not splunk_unit_stat.stat.exists and
            not splunkforwarder_vendor_unit_stat.stat.exists and
            not splunk_vendor_unit_stat.stat.exists

        - name: Re-check for SplunkForwarder.service
          stat:
            path: "/etc/systemd/system/SplunkForwarder.service"
          register: splunkforwarder_unit_stat_after

        - name: Re-check for splunk.service
          stat:
            path: "/etc/systemd/system/splunk.service"
          register: splunk_unit_stat_after

        - name: Re-check vendor SplunkForwarder.service
          stat:
            path: "/usr/lib/systemd/system/SplunkForwarder.service"
          register: splunkforwarder_vendor_unit_stat_after

        - name: Re-check vendor splunk.service
          stat:
            path: "/usr/lib/systemd/system/splunk.service"
          register: splunk_vendor_unit_stat_after

        - name: Determine available systemd unit
          set_fact:
            systemd_unit_name: >-
              {%- if splunkforwarder_unit_stat_after.stat.exists or splunkforwarder_vendor_unit_stat_after.stat.exists -%}
              SplunkForwarder.service
              {%- elif splunk_unit_stat_after.stat.exists or splunk_vendor_unit_stat_after.stat.exists -%}
              splunk.service
              {%- else -%}
              none
              {%- endif -%}
            systemd_unit_path: >-
              {%- if splunkforwarder_unit_stat_after.stat.exists -%}
              /etc/systemd/system/SplunkForwarder.service
              {%- elif splunkforwarder_vendor_unit_stat_after.stat.exists -%}
              /usr/lib/systemd/system/SplunkForwarder.service
              {%- elif splunk_unit_stat_after.stat.exists -%}
              /etc/systemd/system/splunk.service
              {%- elif splunk_vendor_unit_stat_after.stat.exists -%}
              /usr/lib/systemd/system/splunk.service
              {%- else -%}
              none
              {%- endif -%}
            systemd_unit_exists: >-
              {{
                (
                  splunkforwarder_unit_stat_after.stat.exists
                  or splunk_unit_stat_after.stat.exists
                  or splunkforwarder_vendor_unit_stat_after.stat.exists
                  or splunk_vendor_unit_stat_after.stat.exists
                ) | bool
              }}

        - name: Reload systemd daemon
          shell: "systemctl daemon-reload"
          changed_when: false
          failed_when: false
          when: systemd_unit_exists

        - name: Enable discovered systemd unit
          shell: "systemctl enable {{ systemd_unit_name }}"
          register: enable_unit_result
          changed_when: false
          failed_when: false
          when: systemd_unit_exists and systemd_unit_name != "none"

        - name: Start discovered systemd unit
          shell: "systemctl restart {{ systemd_unit_name }}"
          register: systemd_start_result
          changed_when: false
          failed_when: false
          when: systemd_unit_exists and systemd_unit_name != "none"

        - name: Start Splunk directly if no systemd unit found
          shell: "timeout 180 {{ splunk_bin }} start --accept-license --answer-yes --no-prompt"
          register: start_result
          changed_when: false
          failed_when: false
          when: not systemd_unit_exists and upgrade_needed

        - name: Verify installed version
          shell: "{{ splunk_bin }} version 2>&1 | grep -Eo '[0-9]+\.[0-9]+\.[0-9]+' | head -n 1"
          register: verify_out
          changed_when: false
          failed_when: false

        - name: Save verified version
          set_fact:
            version_verified: "{{ verify_out.stdout | default('') | trim }}"

        - name: Check splunk process
          shell: "pgrep -fa splunkd || true"
          register: splunk_process
          changed_when: false
          failed_when: false

        - name: Check discovered systemd unit status
          shell: "systemctl status {{ systemd_unit_name }} 2>&1 || true"
          register: systemd_status_out
          changed_when: false
          failed_when: false
          when: systemd_unit_exists and systemd_unit_name != "none"

        - name: Set result facts
          set_fact:
            splunk_running: "{{ (splunk_process.stdout | default('') | trim | length) > 0 }}"
            upgrade_success: "{{ (not upgrade_needed) or ((version_verified == target_version) and ((splunk_process.stdout | default('') | trim | length) > 0)) }}"
            host_failed: "{{ not ((not upgrade_needed) or ((version_verified == target_version) and ((splunk_process.stdout | default('') | trim | length) > 0))) }}"

        - name: Set failure reason
          set_fact:
            failure_reason: >-
              Validation failed:
              verified_version="{{ version_verified }}"
              process_found="{{ (splunk_process.stdout | default('') | trim | length) > 0 }}"
              systemd_unit_name="{{ systemd_unit_name }}"
              systemd_unit_path="{{ systemd_unit_path }}"
          when: host_failed

      rescue:
        - name: Mark host failed after exception
          set_fact:
            host_failed: true
            failure_reason: "Upgrade workflow exception on {{ inventory_hostname }}"

      always:
        - name: Collect final process state
          shell: "pgrep -fa splunkd || true"
          register: splunk_process_status
          changed_when: false
          failed_when: false
          ignore_errors: true

        - name: Collect final discovered systemd unit status
          shell: "systemctl status {{ systemd_unit_name }} 2>&1 || true"
          register: final_systemd_status
          changed_when: false
          failed_when: false
          ignore_errors: true
          when: systemd_unit_exists and systemd_unit_name != "none"

        - name: Recalculate final running state
          set_fact:
            splunk_running: "{{ (splunk_process_status.stdout | default('') | trim | length) > 0 }}"

        - name: Final summary
          debug:
            msg: >-
              Host={{ inventory_hostname }}
              CurrentVersion={{ current_version }}
              VerifiedVersion={{ version_verified }}
              UpgradeNeeded={{ upgrade_needed }}
              Running={{ splunk_running }}
              UpgradeSuccess={{ upgrade_success }}
              Failed={{ host_failed }}
              SystemdUnitExists={{ systemd_unit_exists }}
              SystemdUnitName={{ systemd_unit_name }}
              SystemdUnitPath={{ systemd_unit_path }}
              Reason="{{ failure_reason }}"

- name: Generate failed hosts report
  hosts: localhost
  gather_facts: yes
  tasks:
    - name: Build host lists
      set_fact:
        all_hosts_list: "{{ groups['all'] | list | sort }}"
        running_hosts: >-
          {{
            groups['all']
            | map('extract', hostvars)
            | selectattr('inventory_hostname', 'defined')
            | selectattr('splunk_running', 'defined')
            | selectattr('splunk_running', 'equalto', true)
            | map(attribute='inventory_hostname')
            | list
            | sort
          }}
        failed_hosts: >-
          {{
            groups['all']
            | map('extract', hostvars)
            | selectattr('inventory_hostname', 'defined')
            | selectattr('host_failed', 'defined')
            | selectattr('host_failed', 'equalto', true)
            | map(attribute='inventory_hostname')
            | list
            | sort
          }}
        unreachable_hosts: >-
          {{
            groups['all']
            | map('extract', hostvars)
            | rejectattr('reached_host', 'defined')
            | map(attribute='inventory_hostname')
            | list
            | sort
          }}

    - name: Build stopped host list
      set_fact:
        stopped_hosts: "{{ all_hosts_list | difference(running_hosts) | sort }}"

    - name: Build failed-stopped host list
      set_fact:
        failed_stopped_hosts: "{{ failed_hosts | intersect(stopped_hosts) | sort }}"

    - name: Write failed hosts list
      copy:
        content: |
          # Failed Hosts
          # Generated: {{ ansible_date_time.iso8601 }}
          # Total failed hosts: {{ failed_hosts | length }}
          # Failed hosts with Splunk stopped: {{ failed_stopped_hosts | length }}

          {% for host in failed_hosts %}
          {{ host }}
          {% endfor %}
        dest: "./failed_hosts.txt"

    - name: Write stopped failed hosts list
      copy:
        content: |
          # Failed Hosts With Splunk Stopped
          # Generated: {{ ansible_date_time.iso8601 }}
          # Total failed+stopped hosts: {{ failed_stopped_hosts | length }}

          {% for host in failed_stopped_hosts %}
          {{ host }}
          {% endfor %}
        dest: "./failed_stopped_hosts.txt"

    - name: Write complete status report
      copy:
        content: |
          # Splunk UF 10.2.0 Upgrade - Status Report
          # Generated: {{ ansible_date_time.iso8601 }}

          SUMMARY
          =======
          Total hosts processed: {{ all_hosts_list | length }}
          Splunk service STOPPED: {{ stopped_hosts | length }}
          Splunk service RUNNING: {{ running_hosts | length }}
          Failed hosts: {{ failed_hosts | length }}
          Failed hosts with Splunk stopped: {{ failed_stopped_hosts | length }}
          Unreachable/Unknown: {{ unreachable_hosts | length }}

          FAILED HOSTS ({{ failed_hosts | length }}):
          {% for host in failed_hosts %}
          - {{ host }}
          {% endfor %}

          FAILED HOSTS WITH SPLUNK STOPPED ({{ failed_stopped_hosts | length }}):
          {% for host in failed_stopped_hosts %}
          - {{ host }}
          {% endfor %}

          STOPPED HOSTS ({{ stopped_hosts | length }}):
          {% for host in stopped_hosts %}
          - {{ host }}
          {% endfor %}

          RUNNING HOSTS ({{ running_hosts | length }}):
          {% for host in running_hosts %}
          - {{ host }}
          {% endfor %}

          UNREACHABLE/UNKNOWN ({{ unreachable_hosts | length }}):
          {% for host in unreachable_hosts %}
          - {{ host }}
          {% endfor %}
        dest: "./splunk_upgrade_status_report.txt"

    - name: Display summary
      debug:
        msg: |
          === SPLUNK UPGRADE REPORT ===
          Total hosts: {{ all_hosts_list | length }}

          STOPPED: {{ stopped_hosts | length }}
          RUNNING: {{ running_hosts | length }}
          FAILED: {{ failed_hosts | length }}
          FAILED_AND_STOPPED: {{ failed_stopped_hosts | length }}
          UNKNOWN: {{ unreachable_hosts | length }}

          Files created:
          - ./failed_hosts.txt
          - ./failed_stopped_hosts.txt
          - ./splunk_upgrade_status_report.txt Yes — the images confirm your latest requirement is valid.
