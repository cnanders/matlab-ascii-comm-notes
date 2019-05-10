# matlab-ascii-comm-notes
Notes on sending and receiving ASCII data packets with MATLAB: 

- fprintf and fscanf vs.
- fwrite and fread vs.
- write and read

# ASCII

Recall that ASCII is a 8-bit character system (256 characters).  For example, the decimal representation of the “carriage return” character is 13, which is 0x0D in hex, or 00001101 in binary. 

# Terminator Characters

MATLAB must always send whatever terminator characters the serial device expects.  When matlab reads data back, it needs to strip any terminator characters sent from the device 

# `fprintf`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

 `fprintf(x, cCommand)` does a few non-obvious things 
 
 1. It formats the command as %s\n (optionally you can provide a format), E.g.:

```matlab
cCommand = 'go' % becomes 'go\n'
```

 2. For serial port, TCPIP, UDP, and VISA-serial objects, `fprintf` replaces all occurrences of \n in the formatted cmd with the Terminator property value. 

 - if `Terminator` = `LF` “line feed”, the decimal representation sent to the device is 10, 
 - if `Terminator` = `CR` “carriage return”, the decimal representation sent to the device is 13
 

 3. It converts each character of the formatted command into an `int8` while taking special care to convert special sequences like “\n” for “line feed” and “\r” for “carriage return” into their correct ASCII values. E.g.:

```matlab
cCommand = 'go' % ASCII
% formatted to 'go\n' (by default)
% the decimal representation of 'g' is 103
% the decimal representation of 'o' is 111
% the decimal representation of '\n' (special “line feed“ sequence) is 10
% the decimal representation of the formatted ASCII data is [103 111 10]
% The result is converted to binary for data transmission [1100111 1101111 0001010]
```
 
## Warning

If the `Terminator` of the serial device is not “line feed” (10 decimal, 0x0A hex), `fprintf` **will not work** unless it is provided with a format that includes the correct `Terminator`.


# `fscanf`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

fscanf() is a nice utility because while it is reading, it reads `bytesAvailable` bytes continuously (while blocking) until it reads the `Terminator` byte.

## `fwrite`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

This is similar to `fprintf` except it writes binary data
- it does no internal formatting of provided data
- it does not append terminator bytes.
 
 If one wishes to manually build data packets, `fwrite` should be used

As mentinoed previously, `fprintf` and `fscanf` do some magic with replacing \n by the terminator that can lead to problems if you are not careful and 100% aware of how they work. 


## Example

Assume the serial device terminator is “line feed + carriage return.

- The decimal representation of the “line feed” ASCII code is 10
- The decimal representation of the “carriage return” ASCII code is 13

```matlab
u8Cmd = [uint8(cCmd) 10 13];
fwrite(this.comm, u8Cmd);
```
## `fread`

Support:
- `MATLAB.serial`
- `MATLAB.tcpip`

## BE EXTRA ADVISED

**write()** and **fwrite()** require the matlab data type to be **uint8**.  For example, if you do fwrite(this.comm, [10 28 10 13]) you are sending a list of four *double*, each double requires 8 bytes (64 bit) to transmit, this would transmit 32 Bytes.  If you intened to send four bytes, one byte for 10 28 10 13, need to cast them as uint8.

### BE ADVISED

*fread(), if not supplied with a number of bytes, will attempt to read tcpip.InputBufferSize bytes.*  In general, **never call fread() without specifying the number of bytes** because it will read for tcpip.Timeout seconds

In general, the programmer does not know how many bytes are returned by each command; the programmer does not know a-priori how many bytes to wait for in the input buffer.

If the programmer did the programmer could use a while loop similar to while (this.comm.BytesAvailable < bytesRequired) that polls BytesAvailable and then only issues the fread(this.comm, bytesRequired) once those bytes are availabe. E.g.:

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

Identical to `fwrite` but for `tcpclient` instances. 


# `read`

Support:
- `MATLAB.tcpclient`

Identical to `fread` but for `tcpclient` instances


 
         

         
  

