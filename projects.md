# Coding and other Projects

### tunx
  
A tool for extracting tcp traffic from icmp data field tunneling.
  
<details>
  <summary>README</summary>

<div markdown="1">

> # tunx
> ### Name:
> tunx, tunnel extractor
>
> ### Synopis:
> python3 tunx [-o offset] [input_file] [output_file]
>
> ### Description:
> Extracts ICMP tunneled TCP/IP layers from scapy compatible packet captures.
>
> Looks for tunneled layer in 'data' field of ICMP packet (ICMP.data of Ether/IP/ICMP frame) and extracts to output file as pcap.
>
> ### Options:
>
> **Required:**
> - [input_file]    Capture file to extract from.  Works with scapy compatible capture files.
> - [output_file]   File to write extracted layer to.
>  
> **Optional:**
> - [-o]            Specify byte offset of tunneled layer in data field.
>
> ### Examples: 
> python3 tunx Ping.pcap extract.pcap
> python3 tunx -o 5 sneakers.pcap extract2.pcap
>
> ### Author:
> James Read

</div>

</details>
  
[Repository](https://github.com/jmrweb/tunx)


### feet

A TUI tool for automated scanning and enumeration that is exstensible with 3rd party plugins.

<details>
  <summary>README</summary>

<div markdown="1">

> # feet
> ### Name:
> feet, friendly extensible enumeration tool
>
> ### Synopis:
> python3 feet
>
> ### Description:
> An application for automated scanning and emumeration using a mouse driven, text user interface.  It is extensible via third party plugins as definied in the API.  feet is built on Textual, Reconnoitre and Redis.
>
> ### Author:
> James Read

</div>

</details>

[Repository](https://github.com/jmrweb/feet)
