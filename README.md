# opkg-extras
opkg-extras.sh
Originaly proposed by @vgaetera

REFS:
- https://openwrt.org/license
- https://openwrt.org/docs/guide-user/advanced/hotplug_extras

====== Opkg extras ======
{{section>meta:infobox:howto_links#cli_skills&noheader&nofooter&noeditbutton}}

===== Introduction =====
  * This instruction extends the functionality of [[docs:guide-user:additional-software:opkg|Opkg]].
  * Follow the [[docs:guide-user:advanced:opkg_extras#automated|automated]] section for quick setup.

===== Features =====
  * Save, restore and roll back Opkg profiles.
  * Support automatic and custom profiles.
  * Support RWM, overlay and extroot setup.
  * Identify user-removed/installed packages.
  * Restore automatically after firmware upgrade.
  * Upgrade all upgradable packages.
  * Find new configurations.

===== Implementation =====
  * Wrap Opkg calls to provide a seamless invocation method.
  * Rely on [[docs:guide-user:base-system:uci|UCI]] and [[https://github.com/openwrt/openwrt/blob/master/package/base-files/Makefile#L47|backup defaults]] to store and manage profiles.
  * Identify package types with [[man>grep(1)|grep]] and [[man>find(1)|find]], and RWM with [[man>mount(8)|mount]].
  * Support importing [[https://github.com/openwrt/openwrt/blob/master/package/base-files/files/sbin/sysupgrade#L245-L251|user-installed]] packages from [[docs:techref:sysupgrade|Sysupgrade]].
  * Use [[docs:guide-user:base-system:hotplug|Hotplug]] to detect WAN connectivity and trigger profile restore.
  * Utilize lockfiles with [[https://github.com/openwrt/openwrt/blob/master/package/utils/busybox/patches/220-add_lock_util.patch|lock]] to avoid race conditions and loops.
  * Perform [[man>reboot(8)|reboot]] to apply changes after automatic profile restore.
  * Write and read non-interactive logs with [[docs:guide-user:base-system:log.essentials|Syslog]] for troubleshooting.
  * Fetch the default profile to restore and roll back from UCI, unless specified manually.
  * Take precedence for remove over install to avoid package conflicts while restoring profile.
  * Limit the operation scope to save time: remove installed, install missing, upgrade upgradable.
  * Process packages one by one to minimize resource consumption and avoid transaction failure.

===== Commands =====
^ Sub-command ^ Description ^
| ''**init**'' | Initialize Opkg configuration. |
| ''**uci** [<prof>] < <uciprof>'' | Import Opkg profile from stdin or file to UCI. |
| ''**import** [<bakprof>]'' | Import Opkg profile from Sysupgrade backup to UCI. |
| ''**save** [<prof>]'' | Save the current Opkg profile to UCI. |
| ''**restore** [<prof>]'' | Restore Opkg profile:\\ ''auto'' or none - default profile,\\ ''rwm'' - RWM profile,\\ ''custom'' - custom profile. |
| ''**rollback** [<prof>]'' | Roll back Opkg profile:\\ remove user-installed packages,\\ install user-removed packages,\\ skip upgraded packages. |
| ''**upgr** [<pkgtype>]'' | Upgrade packages by type:\\ ''ai'' or none - all installed,\\ ''oi'' - overlay-installed. |
| ''**export** [<pkgtype>]'' | Export packages by type:\\ ''ai'' or none - all installed,\\ ''au'' - all upgradable,\\ ''ri'' - ROM-installed,\\ ''wr'' - RWM-removed,\\ ''wi'' - RWM-installed,\\ ''or'' - overlay-removed,\\ ''oi'' - overlay-installed,\\ ''ur'' - user-removed,\\ ''ui'' - user-installed,\\ ''pr'' - profile-removed,\\ ''pi'' - profile-installed. |
| ''**proc** <cmd> < <pkgs>'' | Process packages from stdin with the given sub-command. |
| ''**reinstall** <pkgs>'' | Reinstall specific packages. |
| ''**newconf** [<path>]'' | Find new configurations. |

===== Instructions =====
Copy the following text and paste it in a SSH or serial console of the device
```
# Configure profile
mkdir -p /etc/profile.d
cat << "EOF" > /etc/profile.d/opkg.sh
opkg() {
local OPKG_CMD="${1}"
local OPKG_UCI="$(uci -q get opkg.defaults."${OPKG_CMD}")"
case "${OPKG_CMD}" in
(init|uci|import|save|restore|rollback\
|upgr|export|newconf|proc|reinstall) opkg_"${@}" ;;
(*) command opkg "${@}" ;;
esac
}

opkg_init() {
uci import opkg < /dev/null
uci -q batch << EOI
set opkg.defaults='opkg'
set opkg.defaults.import='/etc/backup/installed_packages.txt'
set opkg.defaults.save='auto'
set opkg.defaults.restore='auto'
set opkg.defaults.rollback='auto'
set opkg.defaults.upgr='ai'
set opkg.defaults.export='ai'
set opkg.defaults.proc='--force-depends'
set opkg.defaults.reinstall='--force-reinstall'
set opkg.defaults.newconf='/etc'
EOI
}

opkg_uci() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
local OPKG_OPT="${OPKG_OPT:-auto}"
if ! uci -q get opkg > /dev/null
then opkg init
fi
uci -q batch << EOI
delete opkg.'${OPKG_OPT}'
set opkg.'${OPKG_OPT}'='opkg'
$(sed -r -e "s/^(.*)\s(.*)$/\
del_list opkg.'${OPKG_OPT}'.'\2'='\1'\n\
add_list opkg.'${OPKG_OPT}'.'\2'='\1'/")
commit opkg
EOI
}

opkg_import() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
if [ -e "${OPKG_OPT}" ]
then sed -n -r -e "s/\s(overlay|unknown)$/\
\tipkg/p" "${OPKG_OPT}" \
| opkg uci auto
fi
}

opkg_save() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
local OPKG_WR="$(opkg export wr)"
local OPKG_WI="$(opkg export wi)"
local OPKG_UR="$(opkg export ur)"
local OPKG_UI="$(opkg export ui)"
if uci -q get fstab.rwm > /dev/null \
&& grep -q -e "\s/rwm\s" /etc/mtab
then {
sed -e "s/$/\trpkg/" "${OPKG_WR}"
sed -e "s/$/\tipkg/" "${OPKG_WI}"
} | opkg uci init
fi
{
sed -e "s/$/\trpkg/" "${OPKG_UR}"
sed -e "s/$/\tipkg/" "${OPKG_UI}"
} | opkg uci "${OPKG_OPT}"
rm -f "${OPKG_WR}" "${OPKG_WI}" "${OPKG_UR}" "${OPKG_UI}"
}

opkg_restore() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
local OPKG_CONF="${OPKG_OPT}"
local OPKG_AI="$(opkg export ai)"
local OPKG_PR="$(opkg export pr)"
local OPKG_PI="$(opkg export pi)"
grep -x -f "${OPKG_AI}" "${OPKG_PR}" \
| opkg proc remove
grep -v -x -f "${OPKG_AI}" "${OPKG_PI}" \
| opkg proc install
rm -f "${OPKG_AI}" "${OPKG_PR}" "${OPKG_PI}"
}

opkg_rollback() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
local OPKG_CONF="${OPKG_OPT}"
local OPKG_UR="$(opkg export ur)"
local OPKG_UI="$(opkg export ui)"
local OPKG_PR="$(opkg export pr)"
local OPKG_PI="$(opkg export pi)"
if uci -q get opkg."${OPKG_CONF}" > /dev/null
then opkg restore "${OPKG_CONF}"
grep -v -x -f "${OPKG_PI}" "${OPKG_UI}" \
| opkg proc remove
grep -v -x -f "${OPKG_PR}" "${OPKG_UR}" \
| opkg proc install
fi
rm -f "${OPKG_UR}" "${OPKG_UI}" "${OPKG_PR}" "${OPKG_PI}"
}

opkg_upgr() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
case "${OPKG_OPT}" in
(ai|oi) opkg_"${OPKG_CMD}"_type ;;
esac | opkg proc upgrade
}

opkg_upgr_type() {
local OPKG_AI="$(opkg export ai)"
local OPKG_OI="$(opkg export oi)"
local OPKG_AU="$(opkg export au)"
case "${OPKG_OPT::1}" in
(a) grep -x -f "${OPKG_AI}" "${OPKG_AU}" ;;
(o) grep -x -f "${OPKG_OI}" "${OPKG_AU}" ;;
esac
rm -f "${OPKG_AI}" "${OPKG_OI}" "${OPKG_AU}"
}

opkg_export() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
local OPKG_TEMP="$(mktemp -t opkg.XXXXXX)"
case "${OPKG_OPT}" in
(ai|au) opkg_"${OPKG_CMD}"_cmd ;;
(ri|wr|wi|or|oi) opkg_"${OPKG_CMD}"_type ;;
(ur|ui) opkg_"${OPKG_CMD}"_run ;;
(pr|pi) opkg_"${OPKG_CMD}"_uci ;;
esac > "${OPKG_TEMP}"
echo "${OPKG_TEMP}"
}

opkg_export_cmd() {
local OPKG_TYPE
case "${OPKG_OPT:1}" in
(i) OPKG_TYPE="installed" ;;
(u) OPKG_TYPE="upgradable" ;;
esac
opkg list-"${OPKG_TYPE}" \
| sed -e "s/\s.*$//"
}

opkg_export_type() {
local OPKG_INFO="/usr/lib/opkg/info"
local OPKG_TYPE
case "${OPKG_OPT::1}" in
(r) OPKG_INFO="/rom${OPKG_INFO}" ;;
(w) OPKG_INFO="/rwm/upper${OPKG_INFO}" ;;
(o) OPKG_INFO="/overlay/upper${OPKG_INFO}" ;;
esac
case "${OPKG_OPT:1}" in
(r) OPKG_TYPE="c" ;;
(i) OPKG_TYPE="f" ;;
esac
find "${OPKG_INFO}" -name "*.control" \
-type "${OPKG_TYPE}" 2> /dev/null \
| sed -e "s/^.*\///;s/\.control$//"
}

opkg_export_run() {
local OPKG_AI="$(opkg export ai)"
local OPKG_RI="$(opkg export ri)"
case "${OPKG_OPT:1}" in
(r) grep -v -x -f "${OPKG_AI}" "${OPKG_RI}" ;;
(i) grep -v -x -f "${OPKG_RI}" "${OPKG_AI}" ;;
esac
rm -f "${OPKG_AI}" "${OPKG_RI}"
}

opkg_export_uci() {
local OPKG_TYPE
case "${OPKG_OPT:1}" in
(r) OPKG_TYPE="rpkg" ;;
(i) OPKG_TYPE="ipkg" ;;
esac
uci -q get opkg."${OPKG_CONF}"."${OPKG_TYPE}" \
| sed -e "s/\s/\n/g"
}

opkg_proc() {
local OPKG_OPT="${OPKG_UCI}"
local OPKG_CMD="${1:?}"
local OPKG_PKG
while read -r OPKG_PKG
do opkg "${OPKG_CMD}" "${OPKG_PKG}" ${OPKG_OPT}
done
}

opkg_reinstall() {
local OPKG_OPT="${OPKG_UCI}"
opkg install "${@}" ${OPKG_OPT}
}

opkg_newconf() {
local OPKG_OPT="${1:-${OPKG_UCI}}"
find "${OPKG_OPT}" -name "*-opkg"
}
EOF
. /etc/profile.d/opkg.sh

# Configure hotplug
mkdir -p /etc/hotplug.d/online
cat << "EOF" > /etc/hotplug.d/online/50-opkg-restore
OPKG_CONF="init main"
for OPKG_CONF in ${OPKG_CONF}
do if [ ! -e /etc/opkg-restore-"${OPKG_CONF}" ] \
&& lock -n /var/lock/opkg-restore \
&& opkg update
then . /etc/profile.d/opkg.sh
opkg restore "${OPKG_CONF/main}" 2>&1 \
| logger -t opkg
touch /etc/opkg-restore-"${OPKG_CONF}"
lock -u /var/lock/opkg-restore
reboot
break
fi
done
EOF
cat << "EOF" >> /etc/sysupgrade.conf
/etc/hotplug.d/online/50-opkg-restore
EOF
```

===== Examples =====
```
# Save Opkg profile
opkg save

# Restore Opkg profile
opkg update
opkg restore

# Roll back Opkg profile
opkg rollback

# Set up a custom Opkg profile
uci set opkg.defaults.restore="custom"
uci set opkg.custom="opkg"
uci add_list opkg.custom.rpkg="dnsmasq"
uci add_list opkg.custom.rpkg="ppp"
uci add_list opkg.custom.ipkg="curl"
uci add_list opkg.custom.ipkg="diffutils"
uci add_list opkg.custom.ipkg="dnsmasq-full"
uci commit opkg

# Check Opkg log
logread -e opkg

# Upgrade packages
opkg update
opkg upgr
```

===== Automated =====
```
uclient-fetch -O opkg-extras.sh "https://openwrt.org/_export/code/docs/guide-user/advanced/opkg_extras?codeblock=0"
. ./opkg-extras.sh
```
  
===== See also =====

Attended sysUpgrade: ''luci-app-attended-sysupgrades'' or ''auc''
