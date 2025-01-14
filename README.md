# grafana-selinux

Grafana selinux policy module for CentOS 7 and RHEL 7. Recently updated to CentOS Stream 9 - Grafana 10.0.3-1.

At present, from my testing this should be working for all basic functions of Grafana. This hasn't been extensively tested(at present. I will change this when I do) by me and should be considered in an early beta state.

The policy assumes that you used the rpm from Grafana to install it. Thus all the file locations should adhere to the rpm or make sure to change them as needed.


### Untested:
* Anything outside the scope of using it at the most basic level - keep an eye on that AVC log!

### Future considerations:
* Add a grafana_plugin_t label to contain plugins.


## Installation
```sh
# Clone the repo
git clone https://github.com/performancecopilot/grafana-selinux.git

# Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules.
# May need to use full path for grafana.if if not working.
install -Dp -m 0664 -o root -g root grafana.if /usr/share/selinux/devel/include/myapplications/grafana.if

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i grafana.pp

# Add grafana ports
semanage port -a -t grafana_port_t -p tcp 3000

# Restore all the correct context labels
restorecon -RvF /usr/sbin/grafana-* \
		/etc/grafana \
		/var/log/grafana \
		/var/lib/grafana \
		/usr/share/performancecopilot-pcp-app

# Start grafana
systemctl start grafana-server.service

# Ensure it's working in the proper confinement
ps -eZ | grep grafana
```

## How To Compile The Module Locally (Needed before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel setools-console
# Change to the directory containing the .if, .fc & .te files
cd grafana-selinux
make -f /usr/share/selinux/devel/Makefile grafana.pp
semodule -i grafana.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a grafana_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
```
If you get a could not open interface info [/var/lib/sepolgen/interface_info] error. 
Ensure policycoreutils-devel is installed and/or run: `sepolgen-ifgen`

## Removing the policy

* To remove the policy, the added port must first be removed
```sh
sudo semanage port -d -p tcp 3000
```
* Now, to remove the policy run
```sh
sudo semodule -r grafana
```
## Compatibility Notes
Built on CentOS Stream 9 at the time with:
```
selinux-policy-38.1.20-1.el9.noarch
selinux-policy-targeted-38.1.20-1.el9.noarch
selinux-policy-devel-38.1.20-1.el9.noarch
```
