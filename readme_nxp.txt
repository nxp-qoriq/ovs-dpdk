export DPCON_COUNT=3
export DPSECI_COUNT=0
export DPMCP_COUNT=3
export DPBP_COUNT=16
export DPCI_COUNT=0
export DPDMAI_COUNT=0

source /usr/local/dpdk/dpaa2/dynamic_dpl.sh dpmac.3 dpmac.4

dpdk-l2fwd -l 2,3 -- -p 0x3 -T 0




# 1. Kill any running OVS
pkill -9 ovs-vswitchd || true
pkill -9 ovsdb-server || true

# 2. Export env variables
export OVS_RUNDIR=/usr/local/var/run/openvswitch
export OVS_LOGDIR=/usr/local/var/log/openvswitch
export OVS_SYSCONFDIR=/usr/local/etc/openvswitch
export DB_SOCK=$OVS_RUNDIR/db.sock
#NXP - if you want lower memory - reduce the default buffer count
export DPDK_NUM_MBUF=10000

# 3. Cleanup old state

rm -f $OVS_SYSCONFDIR/conf.db
rm -rf $OVS_RUNDIR
rm -rf $OVS_LOGDIR

# 4. Create required directories (IMPORTANT)
mkdir -p $OVS_SYSCONFDIR
mkdir -p $OVS_RUNDIR
mkdir -p $OVS_LOGDIR

# 5. Create OVS database
/usr/bin/ovsdb-tool create \
  $OVS_SYSCONFDIR/conf.db \
  /usr/share/openvswitch/vswitch.ovsschema

# 6. Start ovsdb-server (BACKGROUND)
ovsdb-server \
  --remote=punix:$DB_SOCK \
  --remote=db:Open_vSwitch,Open_vSwitch,manager_options \
  --pidfile=$OVS_RUNDIR/ovsdb-server.pid \
  --detach \
  --log-file=$OVS_LOGDIR/ovsdb-server.log \
  $OVS_SYSCONFDIR/conf.db

# 7. Initialize OVS DB
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait init
 
# 8. Enable DPDK + tuning
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  set Open_vSwitch . other_config:dpdk-init=true
 
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  set Open_vSwitch . other_config:dpdk-socket-mem="2048"
 
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  set Open_vSwitch . other_config:dpdk-lcore-mask=0xf
 
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  set Open_vSwitch . other_config:pmd-cpu-mask=0xf

/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  set Open_vSwitch . other_config:dpdk-extra="--file-prefix=ovs"

# 9. Start ovs-vswitchd (BACKGROUND + crash workaround)
ovs-vswitchd unix:$DB_SOCK \
  --disable-system \
  --pidfile=$OVS_RUNDIR/ovs-vswitchd.pid \
  --unixctl=$OVS_RUNDIR/ovs-vswitchd.ctl \
  --log-file=$OVS_LOGDIR/ovs-vswitchd.log \
  --detach

/usr/bin/ovs-vsctl --db=unix:$DB_SOCK set Open_vSwitch . other_config:vhost-sock-dir=/usr/local/var/run/openvswitch

# 10. Create bridge (netdev datapath)
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  add-br br0 -- set bridge br0 datapath_type=netdev

# 11. Add NETWORK PORTS DPDK ports
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK  --no-wait \
  add-port br0 dpdk0 -- set Interface dpdk0 type=dpdk options:dpdk-devargs=dpni.2 options:n_rxq=1

/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  add-port br0 dpdk1 -- set Interface dpdk1 type=dpdk options:dpdk-devargs=dpni.3 options:n_rxq=1

ovs-vsctl --db=unix:$DB_SOCK --no-wait \
set Interface dpdk0 options:dpdk-promisc=true
ovs-vsctl --db=unix:$DB_SOCK --no-wait \
set Interface dpdk1 options:dpdk-promisc=true

# 12. Add vhost-user ports
rm -f /usr/local/var/run/openvswitch/vhost-user1
rm -f /usr/local/var/run/openvswitch/vhost-user2

/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  add-port br0 vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser

/usr/bin/ovs-vsctl --db=unix:$DB_SOCK --no-wait \
  add-port br0 vhost-user2 -- set Interface vhost-user2 type=dpdkvhostuser

# 13. Verify
ps -ef | egrep "ovsdb-server|ovs-vswitchd"
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK show

ovs-appctl -t /usr/local/var/run/openvswitch/ovs-vswitchd.ctl dpctl/show

/usr/bin/ovs-vsctl --db=unix:$DB_SOCK  list Interface
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK  list Port
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK  get Interface dpdk0 ofport
/usr/bin/ovs-vsctl --db=unix:$DB_SOCK  get Interface dpdk1 ofport

ovs-appctl -t /usr/local/var/run/openvswitch/ovs-vswitchd.ctl dpif-netdev/pmd-rxq-show
ovs-appctl -t /usr/local/var/run/openvswitch/ovs-vswitchd.ctl dpif-netdev/pmd-stats-show
ovs-appctl -t /usr/local/var/run/openvswitch/ovs-vswitchd.ctl dpctl/show
