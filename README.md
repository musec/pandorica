# Pandorica

Documentation for our CTF lab environment, which is a self-contained "lab in a box" for hosting diskless clients in full isolation from other networks. Students get root access within a managed environment, allowing them to muck things up locally but not in a durable way. This setup uses:

* PXE booting for diskless clients
* Read-only client root directories stored in ZFS filesystems and served over NFS
* Customized Kali Linux images for students with pre-configured fake users
* CTF challenges exposed via a local Web server

You can set up your own lab by following the instructions:

1. [Pandorica host configuration](host-config.md)
2. [Kali client customization](kali-customization.md)