# Basic TCP stack idea:
Disclaimer: Note that this is just my idea of what to do following the RFC793 guidelines, there are of course multiple ways possible to do it, and not everything will be correct. I will extend this for milestone4 later.

#### There will roughly be 3 classes of events happening: [RFC793, page 51](https://tools.ietf.org/html/rfc793#page-52):
1. The user will make a call to `socket`, `connect`, `send`... or any other call.
2. We receive stuff from the net in `tcp_rx`.
3. Time out is triggered.
#### Note that although a lot of RFC793 is handling edge cases and other stuff that we aren't really required to handle, but it will still be useful to follow the structure they provide.

## 1. User calls

### `socket()` [RFC793, page 53](https://tools.ietf.org/html/rfc793#page-54)
On a call to `socket`, a new socket will be made and managed by our tcp stack. A socket looks a little like this:
```c
struct tcp_socket {
	  struct list_head list;
	  int sockfd;
	
	  int state;
	  uint16_t sport;
	  uint16_t dport;
	  uint32_t saddr;
	  uint32_t daddr;

	  struct tcb tcb;
	
	  struct subuff_head queue_recv;
	  struct subuff_head queue_send;
	  struct timer* retransmit;

	  pthread_rwlock_t lock;
};
```
- `struct list_head list;` will allow us to add this socket to a linked list. In order to make sure we can find our socket later, we will need a global `list_head` node that allows us to look up our previously made sockets. You can use the macro `LIST_HEAD(name)` for this (defined in linklist.h). After allocating your socket, you can add it to the list by calling
```c
list_add_tail(&<name of your socket>.list, &<name of global list node>);
```
- `int sockfd;` is our socket file descriptor, we can generate this ourself. When another call is made by the user, they will use the sockfd they got previously. We can then loop through our list of sockets by using the code below and find the socket that matches the supplied sockfd.
```c
struct list_head* element;
list_for_each(element, &<name of global list node>) {
    struct tcp_socket* socket = list_entry(element, struct tcp_socket, list);
}
```
- `int state;` describes the state our socket is in. We'll mostly use `CLOSED`, `SYN_SENT` or `CONNECTED`, but there are like 11 different states defined in [RFC793, page 20](https://tools.ietf.org/html/rfc793#page-21).
- `uint16_t sport; uint16_t dport;` are the source and destination ports used. The combination of these 2 is used to identify our socket when we receive stuff from the net. In `tcp_rx` we can find our socket by matching the ports received in the tcp header with our list of sockets using the same code above. The source port can be randomly generated, the destination port is set on a call to `connect()` by the user.
- `uint16_t saddr; uint16_t daddr;` These are the source and destination ipv4 addresses used. They aren't necessary in the struct, but still nice to have. The source address will always be `10.0.0.4` (parsed to a uint32_t). The destination address will be set on a call to `connect()` by the user. It will be the address you put your server on. You should probably put your server on your local ipv4 which you can find by running `ifconfig` in your terminal and then looking for the `inet` field after `enp0s3` or whatever your nic is. Usually it is `10.0.1.xx`.
- `struct tcb tcb;` This is a Transmission Control Block as defined on [RFC793, page 18](https://tools.ietf.org/html/rfc793#page-19). It contains some of the variables necessary for transmitting tcp packets, such as the next sequence number to use, the expected incoming sequence number, the latest acknowledged sequence number and other. All of its members are `uint32_t`.
- `struct subuff_head queue_recv; struct subuff_head queue_send;` are queues of subuffs as defined in `subuff.h`. `queue_send` will contain all the subuffs that may need to be retransmitted. This means every subuff that we have transmitted, but the server has not yet acknowledged that it received them. If the retransmit timer times out, we retransmit subuffs stored in this queue. `queue_recv` is subuffs that have been received on the net that have data that can be retrieved by the user with calls to `recv()` (milestone4).
- `struct timer* retransmit;` is a the timer which triggers a retransmit of the packets in `queue_send`. This timer is defined in `timer.h`. You can set it using the code below. The 3rd parameter in the `timer_add` function is the argument that is passed to the function you pass as 2nd param to `timer_add`. You can simply set the cooldown to anything you want (we just used 5000ms over and over again), but it's probably better to dynamically increase this and stop retransmitting after a while.
```c
socket->retransmit = timer_add(<your cooldown>, &<name of retransmit function>, socket);
// Where your retransmit function looks a little like this:
void* <name of retransmit function>(void* arg) {
    struct tcp_socket* socket = (struct tcp_socket*)arg;
    // Other code
}
```
- `pthread_rwlock_t lock;` is a lock that you probably **should** use whenever you use your socket to ensure mutual exclusion. (This might not be necessary for a simple implementation but I haven't tested)

### `connect()` [RFC793, page 53](https://tools.ietf.org/html/rfc793#page-54)
Whenever connect is called we do the following things:
1. We lookup our socket in the list of sockets. If we find a socket that exist, then:
2. Set up the destination address and port in the socket struct by using the sockaddr struct supplied as argument to the `connect` call.
3. Send a SYN segment. (Which also sets the socket state to SYN_SENT).
4. Connect is a blocking call (unless the socket is in a nonblocking mode, which we dont implement), so wait until the state is no longer SYN_SENT (either failed or connected).
5. Return 0 if connected properly (or return your errno in case of any errors that may have happened).

Your send SYN function will allocate a new subuff, reserve the entire buffer, and push the TCP header. Use the fields in the socket struct and the TCB to set the header fields.
Make sure you update the socket state and the next sequence number in the TCB after transmitting. Instead of directly calling your transmit function, you'll likely want to queue the subuff to `queue_send` by using `sub_queue_tail()` as defined in `subuff.h`. Furthermore, you'll have to create a retransmit function and a timer that calls your retransmit function. Your retransmit function will then retransmit the same buffer you had previously stored in the queue, and also reset the timer to keep doing this indefinitely. Note that when you `ip_output` a sub, it'll change the pointers into the buffer data, but not actually mess up the data. You can use `sub_reset_header()` to resolve this. `sub_peek()` is used to get the next entry in the queue. These functions are all defined in `subuff.h`.

#### Note:
Normally this structure is used to ensure that packets are retransmitted in case they are lost. However, in our case, the first packet we send will fail to send because there is not yet a route entry in the ARP stuff. Luckily, this means that the server will never receive our SYN packet, and we won't get an ACK response. Coincidentally this will just cause the retransmit function to be triggered and then the second attempt will succeed :).

### `send()` [RFC793, page 55](https://tools.ietf.org/html/rfc793#page-56)
We can now use the infrastructure to transmit our data, and to queue it for retransmission. On `send()`, we just allocate a `subuff` that accommodates for `min(len, 536)` bytes of data. 536 bytes is the default TCP payload length to avoid ip fragmentation and what not. In case the user wants to transmit more than 536 bytes, we allocate multiple `subuff`s. Each packet gets transmitted and queued for retransmission. Note that every packet now has its ACK flag set to 1.

### `recv()` [RFC793, page 58](https://tools.ietf.org/html/rfc793#page-58)
This is where `queue_recv` comes in. Every packet containing data that is received by our `tcp_rx()` function gets queued to the subuff. Our `recv()` function just loops through the packets in the queue and `memcpy`s the data in the payload of the `subuff` to the user's supplied buffer. Make sure not to overwrite the user's receive length. If all the data from a `subuff` is read, the `subuff` gets dequeued and freed. If only half of the `subuff` is read before the user buffer fills up, we decrease `subuff->dlen` and increase `subuff->paylaod` in order to not read the same information twice the second time around.

### `close()` [RFC793, page 60](https://tools.ietf.org/html/rfc793#page-60)
Very similar to `send`, we just transmit and queue a FIN packet for retranmission. Sending a FIN sets the socket state to FIN_WAIT_1 We then return and let the TCP stack handle the handshake behind the scenes.

## 2. We receive stuff in `tcp_rx()`: The dreaded [SEGMENT ARRIVES (RFC793)](https://tools.ietf.org/html/rfc793#page-65)
Why do we keep retransmitting indefinitely? Well, we don't. Before we set the retransmit timer we check if there is even a buffer in the queue to retransmit. If there isn't, then there is no need to create a new timer and we can just return. So our implementation stops retransmitting buffers when they get removed from the send queue. How? Well this lies at the heart of the TCP protocol. TCP ensures a receiver has received a sender's packets by responding to the packets with an acknowledgement sequence number. This ACK sequence number will be the latest sequence number they received from the sender, incremented by 1. Therefore, when we, as sender, receive an ACK_SEQ, we can go through our `queue_send` and remove all the subuffs that have a sequence number lower than the ACK_SEQ we received. This will in turn cause the retransmission to stop (unless we filled the queue with new unACKed packets). The `tcp_rx()` routine will look something like this:
1. Initialize our TCP header struct. It might also be useful to set the relevant `subuff` member variables (dlen, payload, seq, end_seq). We can use these in our queues to easily determine whether they contain data / can be ACKed.
2. Find the socket we use by using the ports in the TCP header.
3. Follow [SEGMENT ARRIVES (RFC793)](https://tools.ietf.org/html/rfc793#page-65) very closely ;)
#### In order to just make the 3-way handshake work, and not handle any of the errors/edge cases, you can condense the stuff in RFC793 to something like this:
1. Make sure the socket state is SYN_SENT.
2. If the ACK flag is set, make sure the ACK_SEQ is valid.
3. In case the RESET flag is set, close socket, free subuff and return.
4. In case the SYN flag is not set, free the subuff and return.
5. If the ACK flag is set, dequeue all subuffs in `queue_send` that have thereby been acknowledged.
6. Update our socket state to CONNECTED.
7. Send an ACK packet with their incremented SEQ as ACK_SEQ.
8. Free the subuff and return.
#### A general receive packet processing function would then go like this (This is all roughly RFC793):
1. Follow the procedure above if the state is SYN_SENT
2. Verify `SEG.SEQ == RCV.NXT`. Note that regularly you would check that SEG.SEQ falls inside the receive window, but we do not have to implement out of order receives.
3. In case the RESET flag is set, close socket, free subuff and return.
4. In case the SYN flag is not set, free the subuff and return. (We should already be connected)
5. If the ACK flag is not set, free the subuff and return. (Should always be 1 after connecting)
6. Remove acknowledged packets from the retransmission queue up to ACK_SEQ.
7. Free subuff and return if ACK_SEQ duplicate (`< SND_UNA`) or invalid (`> SND_NXT`).
8. If the state is FIN_WAIT_1 and the retransmission queue is empty, then our FIN (being the last packet that was in the queue) must have been acknowledged. Set the state to FIN_WAIT_2.
9. If the state is TIME_WAIT and the retransmission queue is empty, then we must have received a server FIN retransmission. Just send an ACK.
10. If the PSH flag is set, or the packet has data, queue it to the recv queue for our user. ACK the data with `ACK_SEQ += data_len`. Increase `subuff->refcount` in order to not free it at the end of this routine.
11. If the FIN flag is set, that must then be the 3rd step of the closing handshake. ACK it with ++ACK_SEQ. If the state was FIN_WAIT_2, go to TIME_WAIT state and wait 2 minutes before clearing and freeing the socket. If the state was TIME_WAIT, reset the timeout.
12. Free the sub and return.

## 3. A timeout is triggered [RFC793, page 76](https://tools.ietf.org/html/rfc793#page-77)
This is all handled by the timer stuff really. Make sure you have retransmit functions that are called when a retransmission timer times out.
According to RFC793 you should also have a user timeout, which resets everything if the user does not use the socket for a certain time. I don't think it's really necessary to implement this for our purposes, but you can of course do so. (I think just reset the timer everytime a user function is called)
