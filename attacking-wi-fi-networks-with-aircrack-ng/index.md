# Attacking Wi-Fi networks with Aircrack-ng


# Attacking Wi-Fi networks with Aircrack-ng

Before starting, remember that performing any of the attacks explained in this post to networks without previous consent is illegal, this post information is for educational purposes only.

## Introduction

Wi-Fi uses management frames (datagrams are called frames in this context) and data frames. Only data frames are encrypted and injecting them into the network will require a previous association with the AP. This is not necessary for management frames or if we send data frames spoofing some of the clients MAC addresses.

About the hardware you will need, make sure your wireless card support monitor mode. Normally you will need a USB external adapter because internal cards won't allow you to use them in monitor mode but maybe you are lucky! Another thing to note is that in Kali or Parrot modified drivers that allow us to enable monitor mode in certain cards are already in place but these drivers should be installed manually in other distributions. There are cool resources to know what to buy out there, like lists of adapters or [what to look for](https://www.aircrack-ng.org/doku.php?id=compatibility_drivers).

## Capturing traffic

Before explaining anything more, we need to prepare our network card to be able to capture Wi-Fi packages (monitor mode). It is possible to enable monitor mode with:
```
airmon-ng start <iface>
```

After that, you can now start capturing packages in the air. If the last command worked, you should have a new network interface called mon0, that is what you want to use here as interface:
```
airodump-ng -c <channel> -w web_attack <iface>
```

Locking a channel will reduce the noise so it is recommended (`-bssid` option can be used for reducing the noise even more).

## WEP

In this section we will try to learn a bit how WEP works and why it is so vulnerable and then explain how to perform several types of attacks against it.

### How encryption works?

WEP uses a keystream to encrypt the traffic:

<p style="background-color:white; padding-top: 5px;" align="center">
	<img width="75%" height="75%" src="images/wep-crypt-alt.svg" alt="How WEP encryption works">
</p>

### Authentication

The 802.11 specification describes three possible connection states for a client:
1. Not authenticated
2. Authenticated but not associated
3. Authenticated and associated

About how the authentication process work, the 802.11 original standard specified two authentication modes:
- **Open Authentication:** If enabled, the client can just send an authentication request frame and the AP (Access point) will send an authentication result with a success state. After that, the association is performed. This mode can be abuse to perform a "fake auth" and being able to associate to an AP without the key.
- **Shared Key Authentication (SKA):** The AP send a challenge text (128 bytes) and the client has to encrypt it using a Initialization Vector (IV) and the key and send it back to the AP. The AP verifies that the challenge was correct and authenticate the client. The association process can be done after that. In this case, we can abuse that the challenge packages are sent in plaintext to capture the different packages to calculate a keystream we can use to associate to the AP and also inject packages without knowing the key.

### Main flaws

- Weak authentication scheme
- Short initialization vector (IV) and subsequent frecuent reuse.
- Vulnerable to replay attacks
- Weak frame integrity protection thanks to the linearity of the CRC-32 field used as ICV (Integrity check value)
- Low resistance  to related key attacks enabling efficient statistical attacks. For this, it is necessary a certain amount of captured packages with different IVs.

### Attacks

#### Deauthentication Attack

The idea is to send a management packet of type deauth with a MAC address to disconnect from the AP. This can be abused to increase network traffic (for key cracking), get the client connected to an evil twinn, capture the challenge process to get a keystream...

```
aireplay-ng -0 10 -c <client_mac> -a <bssid> <iface>
```

#### ARP Replay Attacks

ARP Replay is the best way of generating more packages in the network to force the creation of new IVs. Once an ARP package is sniffed, it can be re-injected thanks to WEP lack of protection against this type of attacks.

Since ARP use broadcast messages, once we are able to re-inject an ARP package the AP will forward it to all the clients. This will generate new IVs for key cracking.

It is true that, since the network is encrypted, an attacker is not able to read the traffic but since the ARP packages have a fixed payload size of 36 bytes and always have the broadcast address set in the frame header (FF:FF:FF:FF:FF:FF), it is easy to identify them even encrypted.

To perform this attack, we will need to first associtate to the AP. This is how we can perform what is called a "fake auth" abusing open authentication:
```
aireplay-ng -1 15 -a <bssid> -e <ssid> <iface>
```

This also can be used instead to avoid getting deauthenticated by picky APs:
```
aireplay-ng -1 6000 -q 10 -o 1 -a <bssid> -e <ssid> <iface>
```

Now we can start the attack (Don't close the previous terminal!):
```
aireplay-ng -3 -b <bssid> <iface>
```

#### Cracking the key (PWT technique)

Everything we have seen is cool but I know we all want one thing: the AP key. For that we will use `aircrack-ng`.

For the tool to work, it needs network packages. 40 bits keys will need about 5000 IVs to be cracked and 104 bits keys could need aroud ten times more.

The command needs a password size and as you don't know the password length at the time of the attack, a good strategy is first trying with 64 bits and if it fails for more that 10000 IVs try with 128 bits. The default key size used is 128 bits (WEP-104).

Let's see how to start a cracking process over the packages we have captured before:
```
aircrack-ng -n <key_lengh> -e <target_ssid> web_attack1*.cap
```

The tool will start reading the captured packages and if the number of IVs is not enough, it will wait until we capture enough of them. Note that this technique only use captured ARP packages to improve the speed of the cracking process so you will need to launch an ARP replay attack in order to get enough IVs for this to work.

If everything goes well, you should get the AP key in hexadecimal format.

##### KoreK technique

You could want to use the old pre-PWT method to be able to use not only ARP packages when cracking the key. The problem is that you will need much more packages and this method is slower than PWT.

In order to use this method just add the `-K` flag to the `aircrack-ng` command.

#### Clientless WEP cracking

Until now, all the attacks we saw need at least one client connected to the AP in order for them to work.

Let's see how we can get to the cracking step generating ARP packets without clients connected to the network:

1. First authenticate to the AP to get associated to it:
   ```
   aireplay-ng -1 6000 -q 10 -a <bssid> <iface>
   ```
2. Now we can launch the aireplay-ng fragmentation attack, the idea is to be able to get a keystream in order to start forging encrypted packets (make sure to use your wireless adapter MAC):
   ```
   aireplay-ng -5 -b <bssid> -c <source_mac> <iface>
   ```
   This will start capturing package until eventually a data frame is sent from the AP, at this point it will start trying to get a valid keystream. Make sure you are associated with the AP and also to be near enough in order for the attack to work.
3. Time to forge an ARP package now, this will create a package and save it into a file:
   ```
   packetforge-ng -0 -a <bssid> -h <source_mac> -k <ip1> -l <ip2> -y <prga.xor> -w outfile
   ```
   We could use 255.255.255.255 as both ip address and should be fine as most of the AP won't care. The prga thing is the keystream we got before.
4. Now we can use the package generated before to inject it into the network and start flooding the network to increase our IVs collection:
   ```
   aireplay-ng -2 -r <packet_file> <iface>
   ```

#### Bypassing Shared Key Authentication (SKA)

All the authentication we did was using the open authentication method. This is not always enabled so let's see what we can do to bypass the SKA.

The idea is to capture the packets of a client during the authentication phase to get a keystream that we can use to not only to authenticate but also to forge packages. This is the attack steps (make sure you are capturing packages!):

1. Deauthenticate one client.
2. Obtain the keystream from the captured authentication packages. If you are using `airdump-ng` this should happen automatically once a client authenticate to the AP, the keystream will be save in a `.xor` file.
3. Authenticate to the AP using the keystream:
   ```
   aireplay-ng -1 6000 -q 10 -e <ssid> -y <file.xor> <iface>
   ```
4. Start the ARP replay attack, we can both forge an ARP package or wait for a client to send one for us.

#### Attacking the client

Imagine now that you only have access to an AP client but not the AP itself, could we attack the client and get the AP key? Well, looks like this is possible!

Let's see how to perform the so-call Caffe-Latte attack, it is based in the fact that Wi-Fi clients that are roaming periodically send probes looking for an AP they know on every channel. As mutual authentication is not enforced by WEP we can mount a fake AP claiming to be the real one and the client will just connect to it.

Once the client is connected, it will probably send some free ARP packages and if we deauthenticate it to force a new connection we could start collecting packages again but this is slow and we don't want that.

The idea is, once we receive an ARP package from the client we can abuse the weak WEB integrity mechanism in order to flip bits in the package payload to get a valid ARP package that targets the client. Now we can just flood the client with ARP requests and start collecting IVs for free.

Time to start the attack, first of all we have to start `airdump-ng` as usual (locking it to one channel could be a good option since probes are sent to all channels). After that we can start our fake AP with the Caffe-Latte attack enabled with:
```
airbase-ng -c <channel> -W 1 -L -e <ssid> <iface>
```

Now we just have to wait until collecting enough packages to be able to crack the AP key. Another cool flag you can set is `-H` instead of -L, this will use an attack called Hirte (Instead of the Caffe-Latte one) that uses frame fragmentation to speed things up.

## WPA / WPA2

WPA/WPA2 are pretty secure and the attack surface is limited but let's see what can be done.

### Authentication

The WPA/WPA2 authentication use a four-way handshake between the client and the AP, this process permits the mutual authentication between the AP (called Authenticator) and the client (called Supplicant). Both parts must know the PSK (pre-shared key) in order for the process to succeeds.

During the communication the PSK is never sent through the wireless medium. The PSK is only used to generate a PTK (Pairwise Transient Key) that is used as session-only encryption key. This is how the handshake works:

0. The shared key is used to generate the PMK (Pairwise Master Key). This key is 256 bits long and both the client and the AP independently calculate that key combining the PSK and SSID name.
1. The handshake starts now, the AP sends the client a message containing a nonce. In the WPA specification is called ANonce (Authenticator Nonce).
2. The client now generate another nonce, called SNonce (Supplicant Nonce), and builds the PTK concatenating the PMK, both nonces, the MAC addresses of AP and the client and everything is processed using a cryptographic hash function called PBKDF2-SHA1.
3. Now the client sends its SNonce to the AP so now also the AP can build the PTK. This message also contains a MIC (Message Integrity Code) which is used to authenticate the client.
4. The AP replies to the client with a message containing the GTK (Group Temporal Key) used to decrypt multicast and broadcast traffic. Also a MIC is sent to the client (This message is already encrypted).
5. Lastly, the client sends an ACK to end the authentication process.

<p style="background-color:white; padding-top: 5px;" align="center">
	<img width="75%" height="75%" src="images/4-way-handshake.svg" alt="Image of the WPA 4 way handshake">
</p>

If you want to learn more about this part, I recommend [this article](https://www.wifi-professionals.com/2019/01/4-way-handshake).

### Attack

The only way of attacking WPA is brute force, it is not fancy but is what we have. First we will need to capture a 4-way handshake, in order to that we can start capturing packages as we saw earlier and then deauthenticate a user from the target network. That way the client will try to connect back to the AP and we will get the handshake (Also we could just wait for a new client).

There is another method in order to obtain a handshake, `airbase-ng`. Remember when we use this tool for WEP? Well, thanks to how the WPA handshake works we can start a Rogue AP impersonating a SSID without knowing the password for it (The client send us the MIC in the second step so that is enough) and trick a user to connect to give us the handshake:
```
airbase-ng -c <channel> -W 1 -Z 4 -e <ssid> <iface>
```
The `-Z` option is used to specify WPA2 options and the `4` stands for CCMP encryption scheme.

You will know that a handshake was captured because `Airodump-ng` will notify you. Now is time for the cracking part, we could use a dictionay attack or a pure brute force attack. `aircrack-ng` could do the job but it is better if you transform the `.cap` file to a `.hccap` file and use Hashcat for better speeds.

If you prefer a rainbow table attack you can use [Pyrit](https://github.com/JPaulMora/Pyrit). There are [databases already prepared](https://www.renderlab.net/projects/WPA-tables/) to use on the internet if you prefer to skip the database building process.

#### Clientless attack

If you are lucky, maybe the AP is vulnerable to what is called a PMKID attack. Basically, some APs with roaming features enabled would send to the client something called PMKID to make the re-association process faster.

```
PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)
```

This happens during the first part of the handshake so, an attacker, would be able to capture the PMKID to crack the PSK by just trying to associate to the AP without the interaction of any user. I won´t cover how to do this here because you need some extra stuff but you can find everything you need to know in [this hashcat article](https://hashcat.net/forum/thread-7717.html).

## WPS

Wireless Protected Setup (WPS) was designed as a simple and secure method to setup a protected wireless network. WPS provides three different settup alternative methods:

- Push-Button-Connect
- Internal-Registrar
- External-Registrar

The first two methods would need pyhisical or web interface access to the AP but the third option only requires the client to know a 8 digits number PIN.

### Brute Force?

Normally, brute forcing an 8 digits number will require testing for $10^8$ (=100.000.000) combinations but the actual form of authentication used by WPS reduces this number.

The WPS PIN is divided into two halves of 4 digits each. The last digit of the second half is a checksum meaning it is always calculated from the other digits. The authentication process works like this:
1. Both AP and client initialize keys and internal state.
2. Client provides first half of the PIN.
3. Client provides second half of the PIN.
4. AP sends network configuration, including the network password.

The AP will send a NACK packet, terminating the process, if the step 2 or 3 fail. That allow us to perform a pretty efficient brute force attack:

1. We start brute forcing the first half of the PIN incrementing the number to try each time.
2. Once we get to the third step of the authentication process we can repeat the process again with the second half of the PIN.
3. Enjoy your network access!

With this approach, we reduced the number of combinations to try to 20.000 (=$10^4$ + $10^4$). If we also take into account that the last digit it is just a checksum value, the number of combinations is even smaller: $10^4$ + $10^3$ (=11.000).

There are two tools that can help to exploit this:
- [Reaver](https://code.google.com/archive/p/reaver-wps/)
- [Bully](https://github.com/aanarchyy/bully)

We will be using Bully for the brute force attack and a tool called Wash included in Reaver that will allow us to detect vulnerable APs. To start looking for vulnerable APs let's launch Wash:
```
wash -i mon0
```
This command also will show if the AP has blocked WPS access due to internal anti-bruteforce protection (This is the major limitation of this attack). In order to start the attack just execute:
```
bully -b <bssid> <interface>
```
The AP could block WPS after certain failed attempts, to try to avoid this we could add more delay between tries (But this is not perfect). There is an attack called [Pixie Dust](https://www.quora.com/How-does-a-WPS-pixie-attack-work), it is offline and if the AP is vulnerable you could get the PIN in just minutes thanks to a problem with the random generation of nonces used. Bully offers this attack too with the `-d` flag so use it if possible to improve your chances.


