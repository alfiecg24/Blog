---
title: "What I learned from analysing DFU and checkm8"
date: 2022-10-25
---

I am on school holidays at the moment, so I have a lot of time on my hands. Today I decided to take a deep dive into the USB stack of the iPhone's DFU mode, with the main goal of understanding how the checkm8 vulnerability works.

It may not be very readable, but to be honest I just did this to note down everything I discovered. I don't expect anyone to read this, but if you do, I hope it helps you!

*Disclaimer: I did a lot of this research using the leaked iBoot and SecureROM source code - and I obviously cannot include that here. If you have a copy of the source code, take a look at `usb_core.c` and `usb_dfu.c` for the relevant code.

# DFU initialisation

When DFU is first initialised, SecureROM allocates an IO buffer for image transfers, and fill the buffer up with zeros incase any data is left there from previous DFU cycles. After that, SecureROM will setup and register a USB interface to handle requests and image transfers.

# USB transfers
## Setup phase

When the setup phase of a transfer begins, the SecureROM will create a new variable that holds the value of the request's `wIndex`. This value will be used to match the request to the correct USB interface, as all registered interfaces are in an array. After creating this variable, three conditions must be met before processing the USB request.

* the index number is greater than or equal to 0
* the index number is less than the number of total registered interfaces
* the interface that corresponds to the index has a valid request handling function

If all of these conditions are met, the main DFU code hands off the request to the USB interface - which has its own request handling function. This function takes two parameters - a pointer to the request being handled, and a pointer which will be updated with the address of the IO buffer allocated at the start.

If the request is a DFU_DNLOAD request, then two conditions are checked. If the value of `wLength` is 0, that would signal the end of the transfer, so the function returns the 0 `wLength` and does not update the buffer pointer parameter. If `wLength` is not 0, then the code first checks that it is not greater than the size of the actual IO buffer.

If both of these checks are passed, the pointer passed as a parameter is updated with the address of the IO buffer. After this, the function returns the `wLength` back to the main DFU code. The result is once again checked if it is a 0-length packet. If it isn't, then the SecureROM makes sure that the address of the IO buffer was updated correctly, and then updates global variables to tell the code how much data is being transmitted and which interface is handling said transfer. The data phase of the transfer can now begin.

## Data phase
The code begins by verifying that the response length is not 0, before making sure that there is an address for the IO buffer available. After this, it calls a function specifically designed to handle the data phase.

The amount of data left to be transferred and the amount of data to copy into the IO buffer are calculated, before the data that has been received over USB is copied into the IO buffer. After this, the IO buffer is expanded by however much data was copied over.

After this, the function checks if either all the data has been received or the amount of data received does not equal the maximum packet size. If either of those conditions are true, then it is made sure that the interface number is greater than or equal to 0 + that the interface number is less than the number of registered interfaces + the interface that will handle the request has a valid callback function after the data phase has finished.

This callback is then called, and a zero-length packet is added to the execution queue to signify that the transfer is complete.

Finally, the global variables are all set back to their original reset states. When DFU exits, the IO buffer is freed by the system and a new cycle of DFU can begin.


# The problem
The interface request handler updates the global variable passed as an argument to the address of the IO buffer if there is a valid `wLength`. If we send a setup packet when an image is uploading and then a zero-length packet to complete the transaction, we will skip the data phase all together.

We can then use a USB reset to trigger the SecureROM to try and parse the data that was sent in the request. Because we didn't actually send any data, the parsing will inevitably fail. This means DFU will exit, along with freeing the IO buffer, before starting a new cycle of DFU.

Yet, because parsing never finished, the global variables still have all their values from the previous cycle - meaning the variable holding the address of the IO buffer will still hold a valid address to freed memory.

And there is our use-after-free!