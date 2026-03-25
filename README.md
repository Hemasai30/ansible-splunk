---
- name: Upgrade Splunk UF to 10.2.0 (hardened, no systemctl checks)
  hosts: all
  become: yes
  gather_facts: no
  serial: 25

  vars:
    target_version: "10.2.0"
    splunk_bin: "/opt/splunkforwarder/bin/splunk"

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
        host_failed: false
        failure_reason: ""
        splunk_running: false
        upgrade_success: false
        current_version: "not_installed"
        upgrade_needed: true
        version_verified: ""
        reached_host: true

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

    - name: Set version facts
      set_fact:
        current_version: "{{ current_out.stdout | default('not_installed') | trim | default('not_installed') }}"
        upgrade_needed: "{{ (current_out.stdout | default('')) | trim != target_version }}"

    - name: Show current status
      debug:
        msg: "Host={{ inventory_hostname }} UF={{ current_version }} target={{ target_version }} upgrade_needed={{ upgrade_needed }}"

    - name: Upgrade workflow
      block:
        - name: Stop splunk if upgrading and installed
          shell: "timeout 60 {{ splunk_bin }} stop"
          register: stop_result
          changed_when: false
          failed_when: false
          when: upgrade_needed and current_version != 'not_installed'

        - name: Download RPM for RHEL/Amazon Linux x86_64
          get_url:
            url: "{{ splunk_x86_rpm_url }}"
            dest: "{{ x86_rpm_dest }}"
            mode: "0644"
            timeout: 30
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["x86_64", "amd64"]

        - name: Download RPM for RHEL/Amazon Linux arm64
          get_url:
            url: "{{ splunk_arm_rpm_url }}"
            dest: "{{ arm_rpm_dest }}"
            mode: "0644"
            timeout: 30
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["arm64", "aarch64"]

        - name: Install RPM for RHEL/Amazon Linux x86_64
          package:
            name: "{{ x86_rpm_dest }}"
            state: present
          register: pkg_install_result
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["x86_64", "amd64"]

        - name: Install RPM for RHEL/Amazon Linux arm64
          package:
            name: "{{ arm_rpm_dest }}"
            state: present
          register: pkg_install_result
          when: upgrade_needed and ansible_os_family == "RedHat" and ansible_architecture in ["arm64", "aarch64"]

        - name: Download DEB for Debian/Ubuntu amd64
          get_url:
            url: "{{ splunk_amd_deb_url }}"
            dest: "{{ amd_deb_dest }}"
            mode: "0644"
            timeout: 30
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["x86_64", "amd64"]

        - name: Download DEB for Debian/Ubuntu arm64
          get_url:
            url: "{{ splunk_arm_deb_url }}"
            dest: "{{ arm_deb_dest }}"
            mode: "0644"
            timeout: 30
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["arm64", "aarch64"]

        - name: Install DEB for Debian/Ubuntu amd64
          apt:
            deb: "{{ amd_deb_dest }}"
            state: present
          register: pkg_install_result
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["x86_64", "amd64"]

        - name: Install DEB for Debian/Ubuntu arm64
          apt:
            deb: "{{ arm_deb_dest }}"
            state: present
          register: pkg_install_result
          when: upgrade_needed and ansible_os_family == "Debian" and ansible_architecture in ["arm64", "aarch64"]

        - name: Enable Splunk boot-start
          shell: "{{ splunk_bin }} enable boot-start --accept-license --answer-yes --no-prompt"
          register: boot_start_result
          changed_when: false
          failed_when: false
          when: upgrade_needed

        - name: Start splunk
          shell: "timeout 180 {{ splunk_bin }} start --accept-license --answer-yes --no-prompt"
          register: start_result
          changed_when: false
          failed_when: false
          when: upgrade_needed

        - name: Verify version
          shell: "{{ splunk_bin }} version 2>&1 | grep -Eo '[0-9]+\.[0-9]+\.[0-9]+' | head -n 1"
          register: verify_out
          changed_when: false
          failed_when: false

        - name: Save verified version
          set_fact:
            version_verified: "{{ verify_out.stdout | default('') | trim }}"

        - name: Check splunk status directly
          shell: "{{ splunk_bin }} status 2>&1 || true"
          register: splunk_status_cmd
          changed_when: false
          failed_when: false

        - name: Check splunk process
          shell: "pgrep -fa splunkd || true"
          register: splunk_process
          changed_when: false
          failed_when: false

        - name: Set host result facts
          set_fact:
            splunk_running: "{{ (splunk_process.stdout | default('') | trim | length) > 0 }}"
            upgrade_success: "{{ (not upgrade_needed) or ((version_verified == target_version) and ((splunk_process.stdout | default('') | trim | length) > 0)) }}"
            host_failed: "{{ not ((not upgrade_needed) or ((version_verified == target_version) and ((splunk_process.stdout | default('') | trim | length) > 0))) }}"

        - name: Set failure reason when validation fails
          set_fact:
            failure_reason: >-
              Version/process validation failed.
              verified_version="{{ version_verified }}"
              process_found="{{ (splunk_process.stdout | default('') | trim | length) > 0 }}"
              splunk_status="{{ splunk_status_cmd.stdout | default('') }}"
          when: host_failed

        - name: Fail host explicitly if upgrade validation failed
          fail:
            msg: "{{ failure_reason }}"
          when: host_failed

      rescue:
        - name: Mark host failed from rescued exception
          set_fact:
            host_failed: true
            failure_reason: "Upgrade workflow hit an exception on host {{ inventory_hostname }}"

        - name: Capture splunk CLI status after failure
          shell: "{{ splunk_bin }} status 2>&1 || true"
          register: rescue_splunk_status
          changed_when: false
          failed_when: false

        - name: Capture splunk process after failure
          shell: "pgrep -fa splunkd || true"
          register: rescue_splunk_process
          changed_when: false
          failed_when: false

      always:
        - name: Collect final splunk CLI status
          shell: "{{ splunk_bin }} status 2>&1 || true"
          register: splunk_cli_status
          changed_when: false
          failed_when: false
          ignore_errors: true

        - name: Collect final splunk process state
          shell: "pgrep -fa splunkd || true"
          register: splunk_process_status
          changed_when: false
          failed_when: false
          ignore_errors: true

        - name: Recalculate final running state
          set_fact:
            splunk_running: "{{ (splunk_process_status.stdout | default('') | trim | length) > 0 }}"

        - name: Final host summary
          debug:
            msg: >-
              Host={{ inventory_hostname }}
              CurrentVersion={{ current_version }}
              VerifiedVersion={{ version_verified }}
              UpgradeNeeded={{ upgrade_needed }}
              Running={{ splunk_running }}
              UpgradeSuccess={{ upgrade_success }}
              Failed={{ host_failed }}
              Reason="{{ failure_reason }}"

- name: Generate failed hosts report (control node)
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
          - ./splunk_upgrade_status_report.txt
