# Ultimate goal of this project

To write a FTPS client (FTP with ssl) that is able to communicate with the server with essential functionality ('ls', 'rm', 'rmdir', 'mkdir', 'cp', and 'mv')

# High-Level Approach

* For "rm", "rmdir", "mkdir": These command only requires the control channel to function, therefore it is relatively easy to implement as it is simply send and receive message from the control channel.
* For "ls": This command function the same as "ls" in linux/Mac machines, it requires the client to receive data from the data channel, that it is required to tell the server to enter passive mode and receive the data (list of string with content from command "ls")
* For "cp" and "mv" (when uploading): The client would read the file and send it all, server would receive the file. For mv, the local file should be deleted after the upload is complete (this is where I introduce os library to delete the local file)
* For "cp" and "mv" (when downloading): The client should receive the data of the designated file, the recv call has been set to 8192 bytes per call therefore it is required for me to check if the local file size matches the original file. If yes, end the connection, if not, proceed receiving data.

## Basic Logic of this client program:

1. Parse the given argument
   1. understand the command
   2. detect optional argument, apply changes when feature is enabled
   3. detect the number of parameters and understand which is the path of local file and which is the ftp link (for rm and mv)
2. Crate connection to the control channel
   1. Create socket and connect it to the ip and port provided by user in the FTP url (if port not provided, connect to 21, if username is not provided, send username as anonymous)
   2. Wrap the socket in ssl when the user is not anonymous
   3. Send message with credential and essential command to the server
3. If operation does not need data channel, then construct the message and send it over to server
4. If the operation requires data channel, then tell the server to enter passive mode, send the command and wait for data.

## Challenges Faced

1. Similar to basically everyone, I've encountered the 426 error code, when I saw it and tried to solve it, it was frustrating, but thanks to the help of piazza, this problem gets solved by adding the "data.unwrap()".
2. The sending process works flawlessly after adding unwrap(), but downloading is not easy to fix. The thing is that with normal recv() receives 1024 bytes of data, and even with recv(8192), it only receives 8192 bytes of data per call, which means that the program should receive multiple time to meet completeness. This was fixed by introducing the seek and tell, file object would tell the size of itself, if it is not the same number as what appears in the received message, the continues to write data to the file and "seek" the size for future "tell".
3. Another challenge I faced is typo :)
## Testing

1. Considering this is a relatively small program, I've completed all the test entirely by hand.

###This includes:
1. ./3700ftp ls ftps://wangyizho:[correct password]@ftp.3700.network/
   1. List all the file in the root directory
2. ./3700ftp -v mkdir ftps://wangyizho:[correct password]@ftp.3700.network/something
   1. Test the program with verbose mode is enabled
   2. Make directory named something in the root file
3. ./3700ftp -v rmdir ftps://wangyizho:[correct password]@ftp.3700.network/something
   1. This command should run 2 times
      1. To test the something dir created in the previous command
      2. Test the rmdir when the directory does not exist (should not break but should return something different)
4. ./3700ftp -v rm ftps://wangyizho:[correct password]@ftp.3700.network/codememe.png
   1. Should also run 2 times
      1. Test remove a file with the given path
      2. Test rm when the designated file does not exist (should not break)
5. ./3700ftp -v cp codememe.png ftps://wangyizho:[correct password]@ftp.3700.network/codememe.png
   1. Upload the local file to remote server
6. ./3700ftp -v mv codememe.png ftps://wangyizho:[correct password]@ftp.3700.network/codememe.png
   1. Upload the local file to the remote server and delete the local file
7. ./3700ftp -v cp ftps://wangyizho:[correct password]@ftp.3700.network/codememe.png lol.png
   1. Download the file from remote server
8. ./3700ftp -v cp ftps://wangyizho:[correct password]@ftp.3700.network/codememe.png codememe.png
   1. Download the file from remote server and delete it there
9. ./3700ftp -v cp ftps://wangyizho:[correct password]@ftp.3700.network/doesnotexist/codememe.png lol.png
   1. The program should time out since the destination directory does not exist
10. ./3700ftp -v cp ftps://wangyizho:somethingnotcorrect@ftp.3700.network/doesnotexist/codememe.png lol.png
    1. Expect the program to output the error code when verbose is enabled and exit since the password is not correct
11. ./3700ftp -v cp ftps://notcorrect:notcorrect@ftp.3700.network/doesnotexist/codememe.png lol.png 
    1. Expect the program to output the error code when verbose is enabled and exit since the password and username is not correct

