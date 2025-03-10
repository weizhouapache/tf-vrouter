# contrail-vrouter

Contrail Virtual Router

The Contrail Virtual Router implements the data-plane functionality that allows a virtual interface to be associated
with a [VRF](http://en.wikipedia.org/wiki/Virtual_Routing_and_Forwarding).

The Contrail Virtual Router is distributed under the terms of the BSD 2-Clause License and the GPLv2.

The implementation is split into generic "dp-core" and "dpdk" directories used by
multiple operating systems and OS-specific glue. The "linux" directory contains the
Linux specific code.

The utils directory contains user space applications that can be used
to created interfaces (utils/vif) or display the state of the kernel
module.

# building vrouter.ko and contrail-vrouter-dpdk for a specific OS

1. Contrail-dev-env container process can be followed to build vrouter.ko module and
contrail-vrouter-dpdk binary, which can be found [here](https://github.com/Juniper/contrail-dev-env).

2. The process for tungsten fabric can be found in tf-dev-env repository, which can be found
[here](https://github.com/tungstenfabric/tf-dev-env).

# building using entrypoint.sh

```
curl -Lo entrypoint.sh https://raw.githubusercontent.com/tungstenfabric/tf-container-builder/master/containers/vrouter/kernel-build-init/entrypoint.sh
chmod +x ./entrypoint.sh

echo "nightly" >/contrail_version
git clone https://github.com/weizhouapache/tf-vrouter.git /vrouter_src/

./entrypoint.sh
```

# Snapshot of entrypoint.sh (2023-04-02)

```
#!/bin/bash -x

# these next folders must be mounted to compile vrouter.ko in ubuntu: /usr/src /lib/modules

LOG_DIR=${LOG_DIR:-"/var/log/contrail"}
export CONTAINER_LOG_DIR=${CONTAINER_LOG_DIR:-${LOG_DIR}/vrouter-kernel-build-init}

mkdir -p $CONTAINER_LOG_DIR
log_file="$CONTAINER_LOG_DIR/console.log"
touch "$log_file"
chmod 600 $log_file
exec &> >(tee -a "$log_file")
echo "INFO: =================== $(date) ==================="

echo "INFO: Compiling vrouter kernel module for ubuntu..."
current_kver=`uname -r`
echo "INFO: Detected kernel version is $current_kver"

if [ ! -f "/contrail_version" ] ; then
  echo "ERROR: There is no version specified in /contrail_version file. Exiting..."
  exit 1
fi
contrail_version="$(cat /contrail_version)"
echo "INFO: use vrouter version $contrail_version"

vrouter_dir="/usr/src/vrouter-${contrail_version}"
mkdir -p $vrouter_dir
cp -ap /vrouter_src/. ${vrouter_dir}/
chmod -R 755  ${vrouter_dir}
rm -rf /vrouter_src

mkdir -p /vrouter/${contrail_version}/build/include/
mkdir -p /vrouter/${contrail_version}/build/dp-core
mkdir -p /lib/modules/$current_kver/updates/dkms

echo "INFO: run build for current kernel $current_kver"
cd /usr/src/vrouter-"${contrail_version}"
./utils/dkms/gen_build_info.sh "${contrail_version}" /vrouter/"${contrail_version}"/build
make -d -C . KERNELDIR=/lib/modules/$current_kver/build &>> $log_file
cp vrouter.* /lib/modules/$current_kver/updates/dkms/

kernel_modules=$(ls /lib/modules)
for kver in $kernel_modules ; do
  if [[ $kver != $current_kver ]]; then
    echo "INFO: run builds for kernel $kver"
    mkdir -p /lib/modules/$kver/updates/dkms
    make -d -C . KERNELDIR=/lib/modules/$kver/build &>> $log_file
    cp vrouter.* /lib/modules/$kver/updates/dkms/
  fi
done

depmod -a

echo "INFO: check built modules:"
find /lib/modules/ | grep vrouter
echo "INFO: check vrouter.ko was built for current kernel"
ls -l /lib/modules/$current_kver/updates/dkms/vrouter.ko || exit 1

touch $vrouter_dir/module_compiled

# copy vif util to host
if [[ -d /host/bin && ! -f /host/bin/vif ]] ; then
  /bin/cp -f /usr/bin/vif /host/bin/vif
  chmod +x /host/bin/vif
fi

# remove third-party folder
if [[ -d /root/contrail/third_party ]] ; then
  rm -rf /root/contrail/third_party
fi
```
