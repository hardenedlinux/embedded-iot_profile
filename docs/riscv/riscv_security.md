## Software/Firmware/Hardware security on RISC-V 

[RISC-V](https://riscv.org/) is an open ISA can provide more options to build our free/libre firmware/hardware solution. RISC-V may get us rid of "too many blobs" issues, which [is nearly impossible to be resolved in x86](https://hardenedlinux.github.io/system-security/2017/03/17/debian_hardened_boot.html). RISC-V is in very early phase and there are tons of work haven't done yet. [Priviledged ISA spec](https://riscv.org/specifications/privileged-isa/) is still in "draft" stage( v1.10 for now) and security spec is not open for the public yet( maybe in 2018?). The software/firmware side of RISC-V community is growing so fast in past two years, e.g: glibc/binutils got merged already, upstreaming it into linux kernel might still need a couple of months, coreboot can support it partially. In some aspect, RISC-V may have better opportunity to build-security-in in the 1st place. Both x86 and ARM took years to bring those security mitigations into the hardware support. Plz note that those security mitigation also takes time to being utilized by GNU/Linux. Let's quick review the linux kernel defense as an exmaple:

Security defense       | PaX/Grsecurity      | x86                | armv7          | arm64       | coreboot     |
|:--------------------:|:-------------------:|:------------------:|:--------------:|:-----------:|:------------:|
| Non-executable stack | PAGEEXEC( 2000)     | NX bit(2004)       | XN bit( 2011)  | UXN( 2012)  | ?            |
| ret2usr protection   | KERNEXEC( 2004)     | SMEP( 2011)        | PXN( 2014)     | PXN( 2012)  | SMRR( year?) |
| ret2usr protection   | UDEREF( 2007)       | SMAP( 2014)        | N/A            | PAN( 2019?) |              |
| CFI                  | RAP( 2015)          | CET( 2018?)        | N/A            | PA( 2019?)  |              |

For the detail, plz read [linux kernel mitigation](https://github.com/hardenedlinux/grsecurity-101-tutorials/blob/master/kernel_mitigation.md). In the best case, RISC-V could've avoid some mistakes from what x86 and ARM has made. We've been through the paradigm shift of "Attacking the core" from RING 0 to RING -3 in past deacdes. Although firmware( RING -2) and peripheral have high risk attack surfaces, but we should not forget that RING 0( kernel) is still the gate, which the attacker need to be passed through to reach the "underworld". ["The RISC-V Files: Supervisor -> Machine Privilege Escalation Exploit"](http://blog.securitymouse.com/2017/04/the-risc-v-files-supervisor-machine.html) is the 1st public example about "Attacking the Core" on RISC-V, which is an excellent research done by Don A. Bailey. From defensive side, [hardening the COREs](https://github.com/hardenedlinux/hardenedlinux_profiles/blob/master/slide/hardening_the_core.pdf) is the burden we have to carry and it'd be more easier if RISC-V hardware guys can take security into account from the early phase. 

Anyway, I will update this page about RISC-V security stuff from time to time. Free free to add anything about RISC-V security!

## RING -2
[The RISC-V Files: Supervisor -> Machine Privilege Escalation Exploit](http://blog.securitymouse.com/2017/04/the-risc-v-files-supervisor-machine.html)
