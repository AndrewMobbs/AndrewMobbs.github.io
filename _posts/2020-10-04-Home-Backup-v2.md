# Home data and backup architecture

I've recently improved my home data storage and backups, and thought it worth writing up the architecture.

{% plantuml %}
@startuml
actor "User" as actU
usecase "Consume Media" as UC1
usecase "Process data" as UC2
usecase "Backup data" as UC2a
usecase "Access internet" as UC3

actU --> UC1
actU --> UC2
actU --> UC3
UC2 <.. UC2a : <<extends>>
@enduml
{% endplantuml %}

The bulk of data stored is a library of ripped DVDs and Bluray Discs, there's also network drives for shared data (like family photos) and for each user.

## Risks & threat model

In rough order of risk, the main threats to data are (in my assessment):
* Human error
* Device failure
* Malware (specifically ransomware)
* Disaster (fire, flood, electrical damage, children)

## Mitigations

Human error is mitigated through automated regular backups using incremental snapshots.

Device failure is mitigated through
&emsp;- Automated backups mitigating against laptop or server drive failure
&emsp;- NAS redundancy protecting against drive failure
&emsp;- Two NAS devices mitigating against NAS failure

Disaster is mitigated through
&emsp;- Two NAS devices in separate buildings. (Sufficiently small disasters only.)
&emsp;- Positioning well above any plausible flood level.

Malware is mitigated by use of a secure gateway, providing an append-only API for the backup service and a read-only monitoring service to the main network. It specifically does not expose a file-level interface to the NAS behind it or allow shell access from the house network.

Should the main house network suffer a malware attack, there should be no way for it to spread to the secondary NAS, other than via vulnerabilities in one of the two exposed interfaces.

## Implementation
{% plantuml %}
@startuml
hide empty members
hide <<block>> circle
hide interface circle
allowmixing
'skinparam linetype polyline
skinparam monochrome true 
skinparam shadowing false
skinparam classBackgroundColor #FFFFFF
rectangle house {
together {
    class Laptop <<block>>
    interface 192_168_2_* as "192.168.2.*" <<interface>>
    actor User
'    class Duplicati <<block>>
}

together {
    class "Nvidia Shield TV" <<block>>
    interface 192_168_2_66 as "192.168.2.66" <<interface>>
}

together {
    class "HP CP2025dn" <<block>>
    interface 192_168_2_51 as "192.168.2.51" <<interface>>
    circle "LPD"
}

}

rectangle attic {
together {
    class Server <<block>>
'    class "Restic (Server)" <<block>>
    interface 192_168_2_54 as "192.168.2.54"  <<interface>>
    circle "SSH (Server)"
}

together {
    class Synology as "Synology NAS" <<block>>
'    class "Restic (Synology)" <<block>>
    interface 192_168_2_49 as "192.168.2.49" <<interface>>
    circle "SMB (Synology)"
    circle "NFS (Synology)"
    circle "SSH (Synology)"
}
}
rectangle garage {
together {
    class NanoPiR2S <<block>>
'    class "Restic REST Server" <<block>>
'    class "Drobomon" <<block>>
    interface 192_168_4_100 as "192.168.4.100" <<interface>>
    circle "SSH (NanoPi)"
    together {
        interface 192_168_2_53 as "192.168.2.53" <<interface>>
        circle "/v1/drobomon/*"
        circle Restic_REST
    }
}
together {
    class Drobo5N <<block>>
    interface 192_168_4_50 as "192.168.4.50" <<interface>>
    circle "NFS (Drobo)"
    circle "DroboAdmin"
    circle "SSH (Drobo5N)"
}
together {
    class MacPro as "MacPro 1,1" <<block>>
    interface 192_168_4_102 as "192.168.4.102" <<interface>>
}
actor Admin
}

'Laptop  *--  Duplicati
Laptop "wifi" #-- "192_168_2_*"
Laptop "HMI" #-left-  User
"192_168_2_*" --( "SSH (Server)"
"192_168_2_*"  --( "SMB (Synology)"
"192_168_2_*" --( "SSH (Synology)"
"192_168_2_*" --( "LPD"

"Nvidia Shield TV" "eth0" #-- "192_168_2_66"
"192_168_2_66" --( "SMB (Synology)"

"HP CP2025dn" "eth0" #-- "192_168_2_51"
"192_168_2_51" -right- "LPD"

"Nvidia Shield TV" -[hidden]left- "HP CP2025dn"

Synology "eth0" #-left- "192_168_2_49"
"SMB (Synology)" -- "192_168_2_49"
"NFS (Synology)" -- "192_168_2_49"
"SSH (Synology)" -- "192_168_2_49"
"192_168_2_49" --( Restic_REST
'Synology  *-- "Restic (Synology)"

NanoPiR2S "eth0" #-up- "192_168_2_53"
"Restic_REST" -- "192_168_2_53"
"/v1/drobomon/*" -- "192_168_2_53"
NanoPiR2S "lan0" #-- "192_168_4_100"
"192_168_4_100" -left- "SSH (NanoPi)"
"192_168_4_100" --( "NFS (Drobo)"
"192_168_4_100" --( "DroboAdmin"
'NanoPiR2S *-left- "Restic REST Server"
'NanoPiR2S *-left- "Drobomon"

Server "eth0" #--- "192_168_2_54"
"192_168_2_54" -right- "SSH (Server)"
"192_168_2_54" --( "/v1/drobomon/*"
'Server  *-- "Restic (Server)"
"192_168_2_54" --( "NFS (Synology)"

Drobo5N "eth0" #-left- "192_168_4_50"
"NFS (Drobo)" -- "192_168_4_50"
"DroboAdmin" -- "192_168_4_50"
"SSH (Drobo5N)" -- "192_168_4_50"

MacPro "eth0" #-- "192_168_4_102"
Admin --# "HMI" MacPro
"192_168_4_102" --( "SSH (NanoPi)"
"192_168_4_102" --( "SSH (Drobo5N)"

"NFS (Drobo)" )-- "192_168_4_102" 

@enduml
{% endplantuml %}

### Hardware

The primary NAS is a Synology DS1018+, sitting in the attic of my house, on the gigabit ethernet network and powered via a UPS. It is generally fairly open access within the house network for better usability, though this increases the risk of data loss from malware and human error.

The secondary NAS is a Drobo 5N, which was my old NAS before upgrading to the Synology. It is in the garage, on a private network. The garage is a separate building to the main house, with about 5 metres separation.

The secure gateway is a [FriendlyElec NanoPi R2S](https://www.friendlyarm.com/index.php?route=product/product&product_id=282), which is ideal for the purpose. Very cheap, draws very little power, provides two gigabit interface and just enough CPU and RAM. I have one port connected to the house network, bridged to the garage with a [Ethernet over Powerline adaptor](https://www.netgear.com/support/product/xav5001.aspx), and the other to a private network in the garage that has no other path to the outside world.

### Software

#### Backups

Laptops back up nightly to the Synology NAS using [Duplicati](https://www.duplicati.com/) run as a Windows service. The house server that runs various network and monitoring services backs up to the Synology using [Restic](https://restic.net/). The Synology then backs up the media data, network drives and all the other backups to the secondary NAS, again using Restic.

Both Duplicati and Restic are very good at generating small incremental snapshots, only adding new or modified data and metadata, even at a sub-file level. (For example, a large INBOX file will only need the specific changes added, not the entire file backed up each night.)

The main service running on the NanoPi R2s is the [Restic REST server](https://github.com/restic/rest-server) in append-only mode. This provides a HTTP interface allowing RESTIC to store and retrieve blobs, and accesses the underlying storage on the Drobo over NFS.

#### Monitoring

The other service is a monitor for the Drobo. As it's sitting in the garage I need some way of knowing if it's running happily, or if there's a drive failure or other problem. Drobo provides a proprietary GUI, but handily also dumps [a bunch of XML](https://github.com/droboports/droboports.github.io/wiki/NASD-XML-format) on a TCP port. I [wrote a service](https://github.com/AndrewMobbs/drobomon) that essentially just translates that XML to JSON and provides it via HTTP, and also parses the mStatus field to give a HTTP health check endpoint meeting [a random expired draft RFC](https://tools.ietf.org/html/draft-inadarei-api-health-check-04). This is read by the house server (with nothing more sophisticated than a shell script run from cron), which then uses [IFTTT](https://ifttt.com/) to produce alerts on my phone.

## Administration

One obvious wrinkle is that the append-only backups will mean that the Drobo will eventually fill up. This is further complicated because although Restic has the ability to free space by "pruning" old backup data from the repository, is relatively resource intensive, well beyond the capability of the R2S to run. I hit this problem sooner than I'd anticipated when I decided to redo a large part of my video library on the Synology.

The solution was to bring my ancient (well, 2006 vintage) Mac Pro 1,1 out of storage, install Linux on it, and put it in the garage on the "secure" side of the gateway. It then acts as an administrative interface to the Drobo and the NanoPi R2S. Specifically, it has sufficient RAM to run a Restic prune operation to remove old snapshots. (I had considered using the Mac Pro as a server, but the energy costs of running it 24/7 were indefensible for the performance. I bought a [Khadas VIM3](https://www.khadas.com/vim3) single board computer instead, which amusingly has about equivalent CPU performance at <5% of the power consumption.)
