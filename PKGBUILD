# Maintainer: Piotr Gorski <lucjan.lucjanov@gmail.com>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>

### BUILD OPTIONS
# Set these variables to ANYTHING that is not null to enable them

### Tweak kernel options prior to a build via nconfig
_makenconfig=

### Tweak kernel options prior to a build via menuconfig
_makemenuconfig=

### Tweak kernel options prior to a build via xconfig
_makexconfig=

### Tweak kernel options prior to a build via gconfig
_makegconfig=

# NUMA is optimized for multi-socket motherboards.
# A single multi-core CPU actually runs slower with NUMA enabled.
# See, https://bugs.archlinux.org/task/31187
_NUMAdisable=y

# Compile ONLY probed modules
# As of mainline 2.6.32, running with this option will only build the modules
# that you currently have probed in your system VASTLY reducing the number of
# modules built and the build time to do it.
#
# WARNING - ALL modules must be probed BEFORE you begin making the pkg!
#
# To keep track of which modules are needed for your specific system/hardware,
# give module_db script a try: https://aur.archlinux.org/packages/modprobed-db
# This PKGBUILD will call it directly to probe all the modules you have logged!
#
# More at this wiki page ---> https://wiki.archlinux.org/index.php/Modprobed_db
_localmodcfg=

# Use the current kernel's .config file
# Enabling this option will use the .config of the RUNNING kernel rather than
# the ARCH defaults. Useful when the package gets updated and you already went
# through the trouble of customizing your config options.  NOT recommended when
# a new kernel is released, but again, convenient for package bumps.
_use_current=

### Running with a 1000 HZ tick rate 
_1k_HZ_ticks=

### Do not edit below this line unless you know what you're doing

pkgbase=linux-uksm
# pkgname=('linux-uksm' 'linux-uksm-headers' 'linux-uksm-docs')
_major=4.17
_minor=11
pkgver=${_major}.${_minor}
_srcname=linux-${pkgver}
pkgrel=1
arch=('x86_64')
url="https://github.com/dolohow/uksm"
license=('GPL2')
options=('!strip')
makedepends=('kmod' 'inetutils' 'bc' 'libelf')
#_lucjanpath="https://raw.githubusercontent.com/sirlucjan/kernel-patches/master/${_major}"
_lucjanpath="https://gitlab.com/sirlucjan/kernel-patches/raw/master/${_major}"
_uksm_path="https://raw.githubusercontent.com/dolohow/uksm/master"
_uksm_patch="uksm-${_major}.patch"
_gcc_name="kernel_gcc_patch"
_gcc_rel='20180509'
_gcc_path="https://github.com/graysky2/${_gcc_name}/archive"
_gcc_patch="enable_additional_cpu_optimizations_for_gcc_v8.1+_kernel_v4.13+.patch"

source=("https://www.kernel.org/pub/linux/kernel/v4.x/${_srcname}.tar.xz"
        "https://www.kernel.org/pub/linux/kernel/v4.x/${_srcname}.tar.sign"
        "${_gcc_name}-${_gcc_rel}.tar.gz::${_gcc_path}/${_gcc_rel}.tar.gz"
        "${_uksm_path}/${_uksm_patch}"
         # the main kernel config files
        'config'
         # pacman hook for depmod
        '60-linux.hook'
         # pacman hook for initramfs regeneration
        '90-linux.hook'
         # pacman hook for remove initramfs
        '99-linux.hook'
         # standard config files for mkinitcpio ramdisk
        'linux.preset'
        '0001-add-sysctl-to-disallow-unprivileged-CLONE_NEWUSER-by.patch'
        '0002-Revert-drm-i915-edp-Allow-alternate-fixed-mode-for-e.patch'
        '0003-ACPI-watchdog-Prefer-iTCO_wdt-always-when-WDAT-table.patch'
        '0004-mac80211-disable-BHs-preemption-in-ieee80211_tx_cont.patch')

_kernelname=${pkgbase#linux}
: ${_kernelname:=-uksm} 

prepare() {
    cd ${_srcname}

    ### Disable USER_NS for non-root users by default
        msg "Disable USER_NS for non-root users by default"
        patch -Np1 -i ../0001-add-sysctl-to-disallow-unprivileged-CLONE_NEWUSER-by.patch
    
    ### Fix https://bugs.archlinux.org/task/56711
        msg "Fix #56711"
        patch -Np1 -i ../0002-Revert-drm-i915-edp-Allow-alternate-fixed-mode-for-e.patch
    
    ### Fix https://bugs.archlinux.org/task/56780
        msg "Fix #56780"
        patch -Np1 -i ../0003-ACPI-watchdog-Prefer-iTCO_wdt-always-when-WDAT-table.patch
    
    ### Fix iwd provoking a BUG
        msg "Fix iwd provoking a BUG"
        patch -Np1 -i ../0004-mac80211-disable-BHs-preemption-in-ieee80211_tx_cont.patch
    
    ### Patch source with UKSM
        msg "Patching source with UKSM"
	patch -Np1 -i ../${_uksm_patch}
    
    ### Patch source to enable more gcc CPU optimizatons via the make nconfig
        msg "Patching source with gcc patch to enable more cpus types"
	patch -Np1 -i ../${_gcc_name}-${_gcc_rel}/${_gcc_patch}
    
    ### Clean tree and copy ARCH config over
	msg "Running make mrproper to clean source tree"
	make mrproper
	
	cat ../config - >.config <<END
CONFIG_LOCALVERSION="${_kernelname}"
CONFIG_LOCALVERSION_AUTO=n
END
        
    ### Optionally use running kernel's config
	# code originally by nous; http://aur.archlinux.org/packages.php?ID=40191
	if [ -n "$_use_current" ]; then
		if [[ -s /proc/config.gz ]]; then
			msg "Extracting config from /proc/config.gz..."
			# modprobe configs
			zcat /proc/config.gz > ./.config
		else
			warning "You kernel was not compiled with IKCONFIG_PROC!"
			warning "You cannot read the current config!"
			warning "Aborting!"
			exit
		fi
	fi    

	
	### Optionally set tickrate to 1000 
	if [ -n "$_1k_HZ_ticks" ]; then
		msg "Setting tick rate to 1k..."
		sed -i -e 's/^CONFIG_HZ_300=y/# CONFIG_HZ_300 is not set/' \
                    -i -e 's/^# CONFIG_HZ_1000 is not set/CONFIG_HZ_1000=y/' \
                    -i -e 's/^CONFIG_HZ=300/CONFIG_HZ=1000/' ./.config
	fi
	
	### Optionally disable NUMA for 64-bit kernels only
        # (x86 kernels do not support NUMA)
        if [ -n "$_NUMAdisable" ]; then
            msg "Disabling NUMA from kernel config..."
            sed -i -e 's/CONFIG_NUMA=y/# CONFIG_NUMA is not set/' \
                -i -e '/CONFIG_AMD_NUMA=y/d' \
                -i -e '/CONFIG_X86_64_ACPI_NUMA=y/d' \
                -i -e '/CONFIG_NODES_SPAN_OTHER_NODES=y/d' \
                -i -e '/# CONFIG_NUMA_EMU is not set/d' \
                -i -e '/CONFIG_NODES_SHIFT=6/d' \
                -i -e '/CONFIG_NEED_MULTIPLE_NODES=y/d' \
                -i -e '/# CONFIG_MOVABLE_NODE is not set/d' \
                -i -e '/CONFIG_USE_PERCPU_NUMA_NODE_ID=y/d' \
                -i -e '/CONFIG_ACPI_NUMA=y/d' ./.config
        fi

	### Set extraversion to pkgrel and empty localversion
        sed -e "/^EXTRAVERSION =/s/=.*/= -${pkgrel}/" \
            -e "/^EXTRAVERSION =/aLOCALVERSION =" \
            -i Makefile

	### Don't run depmod on 'make install'. We'll do this ourselves in packaging
	sed -i '2iexit 0' scripts/depmod.sh

	### Get kernel version
	msg "Running make prepare for you to enable patched options of your choosing"
	make prepare

	### Optionally load needed modules for the make localmodconfig
	# See https://aur.archlinux.org/packages/modprobed-db
		if [ -n "$_localmodcfg" ]; then
		msg "If you have modprobe-db installed, running it in recall mode now"
		if [ -e /usr/bin/modprobed-db ]; then
			[[ -x /usr/bin/sudo ]] || {
                        echo "Cannot call modprobe with sudo. Install sudo and configure it to work with this user."
                        exit 1; }
			sudo /usr/bin/modprobed-db recall
		fi
		msg "Running Steven Rostedt's make localmodconfig now"
		make localmodconfig
	fi

	### Running make nconfig
	
	[[ -z "$_makenconfig" ]] ||  make nconfig
	
	### Running make menuconfig
	
	[[ -z "$_makemenuconfig" ]] || make menuconfig
	
	### Running make xconfig
	
	[[ -z "$_makexconfig" ]] || make xconfig
	
	### Running make gconfig
	
	[[ -z "$_makegconfig" ]] || make gconfig
	
	### Rewrite configuration
	yes "" | make config >/dev/null

	### Save configuration for later reuse
	cat .config > "${startdir}/config.last"
}

build() {
  cd ${_srcname}

  make bzImage modules
}

_package() {
    pkgdesc='Linux Kernel and modules with the  UKSM.'
    depends=('coreutils' 'linux-firmware' 'mkinitcpio>=0.7')
    optdepends=('crda: to set the correct wireless channels of your country' 'modprobed-db: Keeps track of EVERY kernel module that has ever been probed - useful for those of us who make localmodconfig')
    backup=("etc/mkinitcpio.d/${pkgbase}.preset")
    install=linux.install

    cd ${_srcname}
    
    # get kernel version
    _kernver="$(make kernelrelease)"
    _basekernel=${_kernver%%-*}
    _basekernel=${_basekernel%.*}

    mkdir -p "${pkgdir}"/{boot,usr/lib/modules}
    make INSTALL_MOD_PATH="${pkgdir}/usr" modules_install
    cp arch/x86/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgbase}"

    # make room for external modules
    local _extramodules="extramodules-${_basekernel}${_kernelname}"
    ln -s "../${_extramodules}" "${pkgdir}/usr/lib/modules/${_kernver}/extramodules"

    # add real version for building modules and running depmod from hook
    echo "${_kernver}" |
        install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modules/${_extramodules}/version"

    # remove build and source links
    rm "${pkgdir}"/usr/lib/modules/${_kernver}/{source,build}

    # now we call depmod...
    depmod -b "${pkgdir}/usr" -F System.map "${_kernver}"

    # add vmlinux
    install -Dt "${pkgdir}/usr/lib/modules/${_kernver}/build" -m644 vmlinux

    # sed expression for following substitutions
    local _subst="
        s|%PKGBASE%|${pkgbase}|g
        s|%KERNVER%|${_kernver}|g
        s|%EXTRAMODULES%|${_extramodules}|g
    "

    # hack to allow specifying an initially nonexisting install file
    sed "${_subst}" "${startdir}/${install}" > "${startdir}/${install}.pkg"
    true && install=${install}.pkg

    # install mkinitcpio preset file
    sed "${_subst}" ../linux.preset |
        install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

    # install pacman hooks
    sed "${_subst}" ../60-linux.hook |
        install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/60-${pkgbase}.hook"
    sed "${_subst}" ../90-linux.hook |
        install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/90-${pkgbase}.hook"
    sed "${_subst}" ../99-linux.hook |
        install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/99-${pkgbase}.hook"     
}

_package-headers() {
    pkgdesc='Header files and scripts to build modules for linux-uksm'
    depends=('linux-uksm')

    cd ${_srcname}
    local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

    install -Dt "${_builddir}" -m644 Makefile .config Module.symvers
    install -Dt "${_builddir}/kernel" -m644 kernel/Makefile

    mkdir "${_builddir}/.tmp_versions"

    cp -t "${_builddir}" -a include scripts

    install -Dt "${_builddir}/arch/x86" -m644 arch/x86/Makefile
    install -Dt "${_builddir}/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

    cp -t "${_builddir}/arch/x86" -a arch/x86/include

    install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
    install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

    # http://bugs.archlinux.org/task/13146
    install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

    # http://bugs.archlinux.org/task/20402
    install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
    install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
    install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

    # add xfs and shmem for aufs building
    mkdir -p "${_builddir}"/{fs/xfs,mm}

    # copy in Kconfig files
    find . -name Kconfig\* -exec install -Dm644 {} "${_builddir}/{}" \;

    # add objtool for external module building and enabled VALIDATION_STACK option
    install -Dt "${_builddir}/tools/objtool" tools/objtool/objtool

    # remove unneeded architectures
    local _arch
    for _arch in "${_builddir}"/arch/*/; do
        [[ ${_arch} == */x86/ ]] && continue
        rm -r "${_arch}"
    done

    # remove files already in linux-uksml-docs package
    rm -r "${_builddir}/Documentation"
    
    # remove now broken symlinks
    find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

    # Fix permissions
    chmod -R u=rwX,go=rX "${_builddir}"

    # strip scripts directory
    local _binary _strip
    while read -rd '' _binary; do
        case "$(file -bi "${_binary}")" in
        *application/x-sharedlib*)  _strip="${STRIP_SHARED}"   ;; # Libraries (.so)
        *application/x-archive*)    _strip="${STRIP_STATIC}"   ;; # Libraries (.a)
        *application/x-executable*) _strip="${STRIP_BINARIES}" ;; # Binaries
        *) continue ;;
        esac
        /usr/bin/strip ${_strip} "${_binary}"
    done < <(find "${_builddir}/scripts" -type f -perm -u+w -print0 2>/dev/null)
}

_package-docs() {
    pkgdesc='Kernel hackers manual - HTML documentation that comes with the linux-uksm.'
    depends=('linux-uksm')
  
    cd ${_srcname}
    local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

    mkdir -p "${_builddir}"
    cp -t "${_builddir}" -a Documentation

    # Fix permissions
    chmod -R u=rwX,go=rX "${_builddir}"
}

pkgname=("${pkgbase}" "${pkgbase}-headers" "${pkgbase}-docs")
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#${pkgbase}}")
    _package${_p#${pkgbase}}
  }"
done

sha512sums=('7fcdb7029c79b2629c59422c6c150d71e465333d5d8893f4b0ecf926bc425d934b26e0d25de35e9cd077769c17cd1f3ff0c25c33af9e31c44230be24c2c6eef3'
            'SKIP'
            'a0f37a9b8dbd11f8ef4450b06afee0a6e5519cb5a5cd78f84896812b007ef645bcb9c733ae9817c24d1f4a4c2114258015abceb5a94c7e08d2bb00531a6f04c7'
            '30fc872a19686bd75b0cfa0614980a41d74f4cd40c9fd2c98f82ab4554ad39ccd7ddace9068f354572c1bdf14eed55c8e6d3390127c1a83ec093cf1487a31a0d'
            'ac06781b993da7fd3d156e2e387870667df9c9f2e7bee8a3f40b4f4dca22a54e2a696dddb2dc765b9e39340a9da93c523972ebddb055d5aadbb80e4eeefedc4d'
            '7ad5be75ee422dda3b80edd2eb614d8a9181e2c8228cd68b3881e2fb95953bf2dea6cbe7900ce1013c9de89b2802574b7b24869fc5d7a95d3cc3112c4d27063a'
            '4a8b324aee4cccf3a512ad04ce1a272d14e5b05c8de90feb82075f55ea3845948d817e1b0c6f298f5816834ddd3e5ce0a0e2619866289f3c1ab8fd2f35f04f44'
            '6346b66f54652256571ef65da8e46db49a95ac5978ecd57a507c6b2a28aee70bb3ff87045ac493f54257c9965da1046a28b72cb5abb0087204d257f14b91fd74'
            '2dc6b0ba8f7dbf19d2446c5c5f1823587de89f4e28e9595937dd51a87755099656f2acec50e3e2546ea633ad1bfd1c722e0c2b91eef1d609103d8abdc0a7cbaf'
            'd7160f9a3560c95736d43bc75601a73d507c55b80bd8466bc60b19209fd5d672312db34572bc813df9df4322906032d55ce744df1c05db4bbde014ae92f6241e'
            'ea157e29d4c7d2121586eb1dbcbbb3b15a3cf0074469fbc957ba3f660f71cfe2825da7d7d58824c22d02abbe7995d32491ec381741c5e7ae83e88f6e14944d6e'
            '8f70e4145486c1567c58663bcd9b6e2e519d21f682103edefd916ec9cd1275802f61685dc0f8255471023fd39591f3bbd7ad9293751f6cd3d33580b664d44b3d'
            'f82aecc333581c08b75014a39504c184724802203551b601354452b18ae80e69b5fc8d14f1eb86f75423e9907e13dbe7dbede8758be6477a42dd85238c229b05')
            
validpgpkeys=(
              '647F28654894E3BD457199BE38DBBDC86092693E' # Greg Kroah-Hartman
             )
