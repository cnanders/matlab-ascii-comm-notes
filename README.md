# matlab-ascii-comm-notes
Notes on sending and receiving ASCII data packets with MATLAB: 

- fprintf and fscanf vs.
- fwrite and fread vs.
- write and read

# ASCII

Recall that ASCII is a 8-bit character system (256 characters).  For example, the base10 representation of the “carriage return” character is 13, which is 0x0D in hex, or 00001101 in binary. 

# Terminator Characters in Data Transmission 

MATLAB must always send whatever terminator characters the serial device expects  When matlab reads data back, it will always need to strip any terminator characters sent from the device 

# `fprintf`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

 `fprintf(x, cCommand)` converts the `char` array `cCommand` to an array of int8 (one int8 per character) and subsequently converts each int8 to 8 bits (e.g., 00110100) to form the data packet. In addition, there is also a subtle thing with fprintf that it formats the provided `char` array as %s\n (or optionally you can provide a format) and replaces instances of \n with the terminator byte.  

## Example

```matlab
cCommand = 'go' % ASCII
% the base10 representation of 'go' is [103 111]
% the terminator byte is appended.  Assuming the terminator is “carriage return“, the base10 data is [103 111 13]
% The result is converted to binary for data transmission [1100111 1101111 0001101]
```
# `fscanf`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

fscanf() is a nice utility because while it is reading, it reads `bytesAvailable` bytes continuously (while blocking) until it reads the terminator byte.

## `fwrite`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

This is similar to `fprintf` except it writes binary data
- it does no internal formatting of provided data
- it does not append terminator bytes.
 
 If one wishes to manually build data packets, `fwrite` should be used

`fwrite` and `fread` (see below), provide full control over what is sent and received. `fprintf` and `fscanf` do some magic with replacing \n by the terminator that can lead to problems if you are not careful and 100% aware of how they work. 


## Example

The base10 representation of the “carriage return” ASCII code is 13.
The base10 representation of the “line feed” ASCII code is 10

```matlab
u8Cmd = [uint8(cCmd) 10 13];
fwrite(this.comm, u8Cmd);
```
## `fread`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

### BE ADVISED

*fread(), if not supplied with a number of bytes, will attempt to read tcpip.InputBufferSize bytes.*  In general, **never call fread() without specifying the number of bytes** because it will read for tcpip.Timeout seconds

In general, the programmer does not know how many bytes are returned by each command; the programmer does not know a-priori how many bytes to wait for in the input buffer.

If we did we could have a while loop similar to while (this.comm.BytesAvailable < bytesRequired) that polls BytesAvailable and then only issues the fread(this.comm, bytesRequired) once those bytes are availabe. E.g.:

```matlab
this.waitForBytesAvailable(12);
this.readBytes(12)
```

An alternate approach, below, is a manual implementatation of what fscanf() does, but for binary data.   As bytes become available, read them in and check to see if the terminator character has been found.  Once the terminator is reached, the read is complete. @return {uint8 1xm}

```matlab
function u8 = readToTerminator(this, u8Terminator)
            
      lTerminatorReached = false;
      u8Result = [];
      idTic = tic;
      while(~lTerminatorReached && ...
              toc(idTic) < this.dTimeout )
          if (this.comm.BytesAvailable > 0)
              
              cMsg = sprintf(...
                  'readToTerminator reading %u bytesAvailable', ...
                  this.comm.BytesAvailable ...
              );
              this.msg(cMsg);
              % Append available bytes to previously read bytes
              
              % {uint8 1xm} 
              u8Val = read(this.comm, this.comm.BytesAvailable);
              % {uint8 1x?}
              u8Result = [u8Result u8Val];
              % search new data for terminator
              u8Index = find(u8Val == u8Terminator);
              if ~isempty(u8Index)
                  lTerminatorReached = true;
              end
          end
      end
      
      u8 = u8Result;
      
  end
  ```


# `write`

Support:
- `MATLAB.tcpclient`

Identical to `fwrite` but for `tcpclient` instances


# `read`

Support:
- `MATLAB.tcpclient`

Identical to `fread` but for `tcpclient` instances


 
         

         
  

