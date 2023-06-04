# DNS Enumeration

## Table of contents
- [DNS Record types](#dns-record-types)
  - [ForwardLookup](#forwardlookup)
- [Query a specific type of DNS Record](#query-a-specific-type-of-dns-record)
- [Trace DNS route](#trace-dns-route)
  - [TTL](#TTL)
- [Name Server](#name-server)
  - [Query another DNS server](#queryanother-dns-server)
  - [/etc/resolv.conf](#/etc/resolv.conf)
- [ReverseLookup](#reverselookup)
- [Nmap Scan reverselookup](#nmap-scan-reverselookup)
- [Sub-Domain Enumeration](#sub-domain-enumeration)
  - [dnsrecon](#dnsrecon)
  - [bash](#bash)
- [Zone Transfer](#zone-transfer)
  - [Scan result targets](#scan-result-targets)
- [Host discovering automation](#host-discovering-automation)
  - [dnsrecon](#dnsrecon)
- [Wildcard entries and Bypass](#wildcard-entries-and-Bypass)
- [Additional dig features](#additional-dig-features)
- [Clear DNS Cache](#clear-dns-cache)
- [Tools](#tools)
- [Websites](#websites)


## DNS Record types <a name="DNS Record types"></a>

- **SOA**: The Start of Authority record contains administrative information about the zone transfers. This record shows the DNS server that contains the best (Authoritative) source of information for that specific domain. The output includes the primary name server, mail address, TTL, and more.

- **NS**: Nameserver records contain the name of the authoritative servers hosting the DNS records for a domain.

- **A**: Also known as a host record, the "a record" contains the IP address of a hostname.

- **AAAA**: IPv6 address.

- **MX**: Mail Exchange records contain the names of the servers responsible for handling email for the domain. A domain can contain multiple MX records.

- **PTR**: Pointer Records are used in reverse lookup zones and are used to find the records associated with an IP address.

- **CNAME**: Canonical Name Records are used to create aliases for other host records.

- **TXT**: Text records can contain any arbitrary data and can be used for various purposes, such as domain ownership verification.

[All record types](https://phoenixnap.com/kb/dns-record-types)

---

## Get an IP of a Hostname - ForwardLookup <a name="Forwardlookup"></a>

```shell
#(subdomain required in most situations)
# Basic host command:
host www.domain.com
# By default, the host command looks for an A record that is waht we want right now (modifyable option with -t)

# Basic dig command:
dig www.domain.com
# tweaked for retreaving only the required info (IP of domain.com A record in that case)
dig www.domain.com +short
# tweaked for retreaving a more extensive output of the required info (IP of domain.com A record in that case)
dig www.domain.com +noall +answer

# nslookup
nslookup www.domain.com
```
---

## Query a specific type of DNS Record <a name="Query a specific type of DNS Record"></a>

```shell
# host:
host -t <record-type> domain.com

# dig:
dig domain.com <record-type> +noall +answer
# variant for querying a specif DNS server ip
dig @<dns_ip> domain.com <record-type> +noall +answer
# Request ALL Record Types availables
dig @<dns_ip> domain.com any +noall +answer

# nslookup:
nslookup -type=<record-type> domain.com
```

---

## Trace DNS route <a name="Trace DNS route"></a>

 Follow the delegation path from the root name servers for the name being looked up.

```shell
dig +tarce +short domain.com
```

**TTL Time to Live** <a name="TTL"></a>

```shell
dig +noall +answer +ttlid a domain.com
```

---

## Name Server <a name="Name Server"></a>

A Name server is used for handling requests related to a domain, and those are the authoritative name servers.
Multiple name server tipically exist for a domain, for many reasons, and all are or should be in many cases sync with each other.
Built as a hierarchy a Main(Master) NS is the one that controll the other distributed NS that ask to the main all the necessary info, and are responsible for Zone tranfer that is actually a request of dump everything that the Main NS have (record etc.). Properly configured only other NS of the same domain are allowed to ask for thos type of procedure. So if the querying client is not properly checked a Zone transfer from a non authorized source could happen. (More later)

Find Name Server:
1. Query the domain NS record
2. Retrieve the IP of the newly founded name server

```shell
# Nameserver Hostname
dig domain.com ns +noall +answer
host -t ns domain.com
nslookup -type=ns domain.com

# IPv4
#
dig <nameserver-hostname> +noall +answer
host <nameserver-hostname>
nslookup <nameserver-hostname>
```

Another trick is to Resolve a domain name directly from the Authoritative DNS Server:

```shell
# find the SOA
dig soa domain.com
host -t soa domain.com
nslookup -type=soa domain.com

# perform another lookup specifying the nameserver
dig @<nameserver-hostname> domain.com
nslookup domain.com <nameserver-hostname>
# And thi will give both info about the domain IP and NS IP
```

**Query another DNS server** <a name="Query another DNS server"></a>

If not NS is given to the `dig` command, dig automatically use the servers listed in the default `/etc/resolv.conf` file. To specify a name server against which the query will be executed, use the `@` (at) symbol followed by the name server IP address or hostname. This option is also available in `nslookup` by adding the ip of  choosen DNS server to query after the target domain, and also with the `host command it works the same way. This is a crucial step when searching for info about a domain that has records and hosts splited.

```shell
dig @<dns-server-ip> domain.com
host domain.com <dns-server-ip>
nslookup domain.com <dns-server-ip>
```

**/etc/resolv.conf** <a name="/etc/resolv.conf"></a>

It's crucial to keep in mind that if you are dealing with a private network or a particular situation in which the the name server is not reachable directly from your DSN configuaretions, you need to manually add the private/custom dns NS to the resolv.conf file in order to be able to comunicate with it properly!
And it have to be added at the beginning before your default nameserver:

```shell
# resolvc.conf
#Generated by NetworkManager
#search isp <--- comment this
nameserver 192.168.50.71 <--- #here 
nameserver 192.168.1.1
nameserver ipv6
```

There is also a resolvconf app that can handle all of this for you, if you don't want to doit manually.

---

## Get the Hostname of an IP - ReverseLookup <a name="ReverseLookup"></a>

```shell
# host:
host <ip>

# dig:
dig -x <ip> +short
# reverse lookup trick - write in reverse order the IP=192.168.23.66 you are about to query and add .in-addr.arpa making the dig comand requieting a PTR record:
dig 66.23.168.192.in-addr.arpa PTR

# nslookup:
# nslookup by default automatically attmpt to query or an A record if a hostname is provided or a PTR record for indeed a reverse lookup to find the domain name at which an IP is pointing
nslookup domain.com
```

## Nmap Scan target range for a reverse DNS lookup <a name="Nmap Scan forwardlookup"></a>

```shell
# -sL: perform only a dns resolution not a proper scan
# --dns-servers: External dns has been added (handle the traffic volume)
 nmap --dns-servers 8.8.8.8,1.1.1.1 -sL 200.12.94.0/24 177.123.45.0/24
```


---

## Sub-Domain Enumeration <a name="Sub-Domain Enumeration"></a>

**dnsrecon** <a name="dnsrecon"></a>

```shell
dnsrecon -d domain.com -D /usr/share/seclists/Discovery/DNS/... -t brt
```

**bash** <a name="bash"></a>

```shell
for i in $(cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt); do host $i.domain.com; done | grep "has address"
```

---

## Zone Transfer <a name="Zone Transfer"></a>

DNS Zone [Wiki](https://en.wikipedia.org/wiki/DNS_zone)

DNS Zone Transfer [Wiki](https://en.wikipedia.org/wiki/DNS_zone_transfer)

```shell
# host
host -l domain.com <nameserver-ip>
# complete info with all the records
host -a -l domain.com <nameserver-ip>

# dig
dig axfr @<nameserver-ip> domain.com
#
dig axfr @<nameserver-ip> domain.com +answer > zonetransfer
cat zonetransfer | grep domain.com | grep IN | awk '{print $1}' | sed 's/\.$//g' | sort -u > livetargets
# if scanning private network add these results in /etc/hosts
# instead of adding multiple host lines just add one host IP with multiple values separeated by space
tr '\n' ' ' < livetargets > add_to_hosts

# dnsrecon
dnsrecon -d domain.com -t axfr

# dnsenum
dnsenum domain.com
```

**Scan result targets** <a name="Scan result targets"></a>

```shell
# append http:// at the beginning of each domain name
sed 's/^/https:\/\//' livetargets > www-livetargets
# or the following command to overwrite the file
sed -i 's/^/https:\/\//' livetargets

# aquatone
cat www-livetargets | aquatone
```

Useful guide [link](https://infinitelogins.com/2020/04/23/performing-dns-zone-transfer/)

---

## Host discovering automation <a name="Host discovering automation"></a>

**dnsrecon** <a name="Forwardlookup"></a>

Internal network discovery

```shell
# SQLite DB file
dnsrecon -n <namserver-ip> -r 192.168.0.0/24 --db target.db
```
#

Just brief example with `host` command, customizable.

```shell
#
for ip in $(cat ip-targets-list.txt); do host $ip; done | grep -v "not found"

#
for i in $(seq 1 256); do host 192.168.200.$i; done | grep -v "not found"
```

---

## Wildcard entries and Bypass <a name="Wildcard entries and Bypass"></a>

A wildcard DNS record is a record in a DNS zone that will match requests for non-existent domain names. [Wiki](https://en.wikipedia.org/wiki/Wildcard_DNS_record)

TO DO

---

### Additional dig features <a name="Additional dig features"></a>

**Force IP transport version**

```shell
# IPv4
dig -4 <normal-query>
# IPv6
dig -6 <normal-query>
```

**Specify Port Number**

```shell
# 
dig -p 666 <normal-query>
```

Use the `+[no]vc` or `+[no]tcp` flag to control TCP or UDP protcols. Please note that all `AXFR` queries always use TCP.

---

### Clear DNS Cache <a name="Clear DNS Cache"></a>

```shell
# Windows
# renew IPv4
ipconfig /release
ipconfig /renew
# flush dns cache
ipconfig /flushdns

# Linux
# Check first if the system is storing DNC cache
systemctl is-active systemd-resolve
# If the response is “active”, DNS caching is taking place. If the output is “inactive” , the caching is disabled
# use the systemd-resolve or resolvectl command with the statistics option to see how many records are in the cache
systemd-resolve --statistics
resolvectl statistics
# 
sudo killall -USR1 systemd-resolve
#
sudo journalctl -u systemd-resolve > dns.txt
#
cat dns.txt 
less dns.txt
# Then to Flush Debian/Ubuntu
sudo systemd-resolve --flush-caches
# other distro
sudo /etc/init.d/nscd restart
# or
resolvectl flush-caches
```
> Linux [source](https://www.howtogeek.com/844964/how-to-flush-dns-in-linux/) and explanation

> [Renew](https://www.cyberciti.biz/faq/howto-linux-renew-dhcp-client-ip-address/) IP address in Linux

---

## Tools <a name="Tools"></a>

- [dnsrecon](https://github.com/darkoperator/dnsrecon)

- [dnsenum](https://github.com/SparrowOchon/dnsenum2)

- [fierce](https://github.com/mschwager/fierce)

- [reverse-scan](https://github.com/amine7536/reverse-scan)

- [Powermad](https://github.com/Kevin-Robertson/Powermad)

- [aquatone](https://github.com/michenriksen/aquatone)



---

## Websites <a name="Websites"></a>

- [DNS Checker](https://dnschecker.org/all-tools.php)

- [MX Toolbox](https://mxtoolbox.com/DnsLookup.aspx)

- [Whatismyipaddress](https://whatismyipaddress.com/ip-hostname)

- [Reverseip](https://reverseip.domaintools.com/)

- [Viewdns](https://viewdns.info/)

- [Google dig](https://toolbox.googleapps.com/apps/dig/)

- [Dnsinspect](https://dnsinspect.com/)

- [Google DNS](https://dns.google/)

- [Dns-history](https://dns-history.whoisxmlapi.com/)

- [DNS Propagation map](https://dnsmap.io/)

- [Propagation](https://dnschecker.org/)
