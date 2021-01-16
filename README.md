# BlueToothSniffer

<h1 align="center">
	<img src="https://i.ibb.co/8Pmwgch/sniff-bluetooth.png" alt="Sniffer">
</h1>

> here is my code : 
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <bluetooth/bluetooth.h>
#include <bluetooth/hci.h>
#include <bluetooth/hci_lib.h>

int main(int argc, char **argv)
{
    inquiry_info *ii = NULL;
    int max_rsp, num_rsp;
    int dev_id, sock, len, flags;
    int i;
    char addr[19] = { 0 };
    char name[248] = { 0 };

    dev_id = hci_get_route(NULL);
    sock = hci_open_dev( dev_id );
    if (dev_id < 0 || sock < 0) {
        perror("opening socket");
        exit(1);
    }

    len  = 8;
    max_rsp = 255;
    flags = IREQ_CACHE_FLUSH;
    ii = (inquiry_info*)malloc(max_rsp * sizeof(inquiry_info));
    
    num_rsp = hci_inquiry(dev_id, len, max_rsp, NULL, &ii, flags);
    if( num_rsp < 0 ) perror("hci_inquiry");

    for (i = 0; i < num_rsp; i++) {
        ba2str(&(ii+i)->bdaddr, addr);
        memset(name, 0, sizeof(name));
        if (hci_read_remote_name(sock, &(ii+i)->bdaddr, sizeof(name), 
            name, 0) < 0)
        strcpy(name, "[unknown]");
        printf("%s  %s\n", addr, name);
    }

    free( ii );
    close( sock );
    return 0;
}
```
> Compile it with ! :
```bash
gcc -o simplescan simplescan.c -lbluetooth
```
> Explanation : The basic data structure used to specify a Bluetooth device address is the bdaddr_t. All Bluetooth addresses in BlueZ will be stored and manipulated as bdaddr_t structures. BlueZ provides two convenience functions to convert between strings and bdaddr_t structures. 
```c

typedef struct {
	uint8_t b[6];
} __attribute__((packed)) bdaddr_t;
```
> `str2ba` takes an string of the form `XX:XX:XX:XX:XX:XX`, where each XX is a hexadecimal number specifying an octet of the 48-bit address, and packs it into a 6-byte bdaddr_t. ba2str does exactly the opposite. 
```c
int str2ba( const char *str, bdaddr_t *ba );
int ba2str( const bdaddr_t *ba, char *str );
```
> Local Bluetooth adapters are assigned identifying numbers starting with 0, and a program must specify which adapter to use when allocating system resources. Usually, there is only one adapter or it doesn't matter which one is used, so passing NULL to hci_get_route will retrieve the resource number of the first available Bluetooth adapter. 
```c
int hci_get_route( bdaddr_t *bdaddr );
int hci_open_dev( int dev_id );
```
- [!] Note : 
  - It is not a good idea to hard-code the device number 0, because that is not always the id of the first adapter. For example, if there were two adapters on the system and the first adapter (id 0) is disabled, then the first available adapter is the one with id 1 .
  
> If there are multiple Bluetooth adapters present, then to choose the adapter with address ``01:23:45:67:89:AB", pass the char * representation of the address to hci_devid and use that in place of hci_get_route :
```c

int dev_id = hci_devid( "01:23:45:67:89:AB" );
```

> After choosing the local Bluetooth adapter to use and allocating system resources, the program is ready to scan for nearby Bluetooth devices. In the example, hci_inquiry performs a Bluetooth device discovery and returns a list of detected devices and some basic information about them in the variable ii. On error, it returns -1 and sets errno accordingly. 
```c
int hci_inquiry(int dev_id, int len, int max_rsp, const uint8_t *lap, inquiry_info **ii, long flags);
```
- hci_inquiry is one of the few functions that requires the use of a resource number instead of an open socket, so we use the dev_id returned by hci_get_route. The inquiry lasts for at most 1.28 * len seconds, and at most max_rsp devices will be returned in the output parameter ii, which must be large enough to accommodate max_rsp results. We suggest using a max_rsp of 255 for a standard 10.24 second inquiry. 
> If flags is set to IREQ_CACHE_FLUSH, then the cache of previously detected devices is flushed before performing the current inquiry. Otherwise, if flags is set to 0, then the results of previous inquiries may be returned, even if the devices aren't in range anymore.
The inquiry_info structure is defined as :
```c

typedef struct {
    bdaddr_t    bdaddr;
    uint8_t     pscan_rep_mode;
    uint8_t     pscan_period_mode;
    uint8_t     pscan_mode;
    uint8_t     dev_class[3];
    uint16_t    clock_offset;
} __attribute__ ((packed)) inquiry_info;
```
> Once a list of nearby Bluetooth devices and their addresses has been found, the program determines the user-friendly names associated with those addresses and presents them to the user. The hci_read_remote_name function is used for this purpose. 
```c

int hci_read_remote_name(int sock, const bdaddr_t *ba, int len, 
                         char *name, int timeout)
```
- hci_read_remote_name tries for at most timeout milliseconds to use the socket sock to query the user-friendly name of the device with Bluetooth address ba. On success, hci_read_remote_name returns 0 and copies at most the first len bytes of the device's user-friendly name into name. On failure, it returns -1 and sets errno accordingly. 
