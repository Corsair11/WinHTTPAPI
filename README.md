# WinHTTPAPI
Windows service that provides simple HTTP JSON API to manage filesystem, execute commands, retrieve info etc.
Installation:
1) Build
2) Create C:\WinHTTPAPI directory
3) Put bin/Release contents into it
4) Launch WinHTTPAPIInstall.bat file (as Administrator)
5) Create WinHTTPAPIAdministrators and/or WinHTTPAPIUsers groups in your Windows system
6) Create and/or add existing Windows users to this group(s)
7) Don't forget to add allow firewall rule (TCP:2950)
8) Try it! (ex. curl http://target_machine:2950/check --ntlm --negotiate -u user:pass)

   # API Description:
   /check - check, is user authorized and API working (return example: {"State":"success","Message":"Access granted","AdditionalInformation":"WinHTTPAPI v1.0.0.0 (WINDOWS-PC)"})
   /permissions - authorized user permissions (WinHTTPAPI User or Admin, return example: {"State":"success","Message":"WinHTTPAPI Admin","AdditionalInformation":"admin"})
   /version - WinHTTPAPI Service version (return example: {"State":"success","Message":"Service version extracted","AdditionalInformation":"WinHTTPAPI v1.0.0.0"})
   /isotime - Target machine time, in ISO format (return example: {"State":"success","Message":"Server time extracted","AdditionalInformation":"2023-10-23T14:34:07"})
   /hostname - Target machine hostname (return example: {"State":"success","Message":"WINDOWS-PC","AdditionalInformation":null})
   /pathinfo - Information about filesystem object (usage: http://target_machine:2950/pathinfo?path=D:\SQLData, return example: {"State":"success","Message":"Path info extracted","AdditionalInformation":"{\"Name\":\"SQLData\",\"FullPath\":\"D:\\\\SQLData\",\"Size\":16777216,\"ObjectType\":1,\"HashCalculated\":false,\"Sha256\":null,\"LastWrite\":\"2023-05-19T09:30:30.6144744+03:00\",\"CreationTime\":\"2023-05-15T12:10:24.4561123+03:00\",\"HumanReadableSize\":\"16 MB\"}"})
   pathinfo method can handle additional parameter calchash=true|false, false by default, if it's file, sha256 fingerprint will also be shown
   /list - list specified directory contents (usage: http://target_machine:2950/list?path=D:\SQLData&calchash=true, return example: {"State":"success","Message":"Object list extracted","AdditionalInformation":"[{\"Name\":\"base.mdf\",\"FullPath\":\"D:\\\\SQLData\\\\base.mdf\",\"Size\":8388608,\"ObjectType\":0,\"HashCalculated\":true,\"Sha256\":\"e3a70d1309d0f9dba66116b3b28d610200764f3038a430e026757812252133ae\",\"LastWrite\":\"2023-06-08T14:17:48.7288719+03:00\",\"CreationTime\":\"2023-05-19T09:30:30.5832319+03:00\",\"HumanReadableSize\":\"8 MB\"},{\"Name\":\"base_log.ldf\",\"FullPath\":\"D:\\\\SQLData\\\\base_log.ldf\",\"Size\":8388608,\"ObjectType\":0,\"HashCalculated\":true,\"Sha256\":\"d26e9db2dcd7fb49b7077db65b3b3b3d3f9d7560b9e34ae490bf97c6ad2c1161\",\"LastWrite\":\"2023-06-08T14:17:48.7288719+03:00\",\"CreationTime\":\"2023-05-19T09:30:30.6144744+03:00\",\"HumanReadableSize\":\"8 MB\"}]"})
   
   IMPORTANT! If target directory contains many large files, calchash=true can take a lot of time
   list method can handle additional parameter (nodirsize=true|false, false by default) count or no directories contents size in list (if listed dir located in HDD and contains hundreds of thousands files, it can take a lot of time)
   
   /mkdir - create directory in a target computer filesystem (usage: http://target_machine:2950/mkdir?path=D:\NewDir, return example: {"State":"success","Message":"Directory created","AdditionalInformation":"{\"Name\":\"NewDir\",\"FullPath\":\"D:\\\\NewDir\",\"Size\":0,\"ObjectType\":1,\"HashCalculated\":false,\"Sha256\":null,\"LastWrite\":\"2023-10-23T14:55:47.3313015+03:00\",\"CreationTime\":\"2023-10-23T14:55:47.3313015+03:00\",\"HumanReadableSize\":\"0 B\"}"})
   
   /rootlist - lists target machine's disks as directories, as /list do it with typical dir (see /list above)
   
   /remotecmd - executes specified command (or program) on target machine (usage - http://target_machine:2950/remotecmd, params must be sent by POST (JSON), see RemoteCmd class
   
   example (usage): curl -X POST -d "{'CmdLabel':'PING', 'CmdObject':'ping', 'CmdArgs':'192.168.5.1 -n 3'}" http://192.168.5.5:2950/remotecmd --ntlm --negotiate -u user:pass
   
   return example: {"State":"success","Message":"Command executed","AdditionalInformation":"{\"ReturnCode\":0,\"StdOut\":\"\\r\\nPinging 192.168.5.1 with 32 bytes of data:\\r\\nReply from 192.168.5.1: bytes=32 time<1ms TTL=64\\r\\nReply from 192.168.5.1: bytes=32 time<1ms TTL=64\\r\\nReply from 192.168.5.1: bytes=32 time<1ms TTL=64\\r\\n\\r\\nPing statistics for 192.168.5.1:\\r\\n    Packets: Sent = 3, Received = 3, Lost = 0 (0% loss),\\r\\nApproximate round trip times in milli-seconds:\\r\\n    Minimum = 0ms, Maximum = 0ms, Average = 0ms\\r\\n\",\"StdErr\":\"\"}"}
   
   note: You can specify CmdWaitTicks param with max of milliseconds you want to "wait" command execution before it will be forcefully killed by WinHTTPAPI service, default - 1 minute (60000ms)
   
   note 2: Remote commands output console encoding hardcoded to 866, you can change it and rebuild
   
   /upload - upload file to target machine

   example (usage): curl -X POST --data-binary @test.exe http://192.168.5.5:2950/upload?path=D:\test.exe --ntlm --negotiate -u user:pass

   return example: {"State":"success","Message":"File accepted","AdditionalInformation":"{\"Name\":\"test.exe\",\"FullPath\":\"D:\\\\test.exe\",\"Size\":151667336,\"ObjectType\":0,\"HashCalculated\":false,\"Sha256\":null,\"LastWrite\":\"2023-10-23T16:33:34.4133789+03:00\",\"CreationTime\":\"2023-10-23T16:27:40.8552124+03:00\",\"HumanReadableSize\":\"144,64 MB\"}"}

   /getfile - downloads file from target machine by path

   usage: http://target_machine:2950/getfile?path=D:\test.exe

   If successful, returns a file itself as attachment (application/octet-stream).

   /copy - copy file or directory inside target machine's filesystem

   example (usage): curl "http://192.168.5.5:2950/copy?src=D:\test.exe&dest=D:\newfile.exe" --ntlm --negotiate -u user:pass

   return example: {"State":"success","Message":"Object has copied","AdditionalInformation":"Source path: D:\\test.exe\r\nDestination path: D:\\newfile.exe"}

   /move - move file or directory inside target machine's filesystem

   usage: same as /copy (see above)

   /remove - remove file or directory inside target machine's filesystem

   example (usage): curl "http://192.168.5.5:2950/remove?path=D:\newfile.exe" --ntlm --negotiate -u

   return example: {"State":"success","Message":"Object has deleted","AdditionalInformation":"Path: D:\\newfile.exe\r\nObject type: Unknown"}

   note: "Unknown" object type according to this method and other filesystem API methods above means that object not exist.

   /logon - list logon users (active sessions at this time)

   example (usage): curl http://192.168.5.5:2950/logon --ntlm --negotiate -u user:pass

   return example: {"State":"success","Message":"Active sessions list extracted","AdditionalInformation":"[{\"UserName\":\"User\",\"ClientName\":\"\",\"ClientIP\":null,\"ConnectionState\":\"Active\",\"SessionID\":1}]"}

   note: this method is useful for remote desktop servers

   /kickuser - force specified user session logoff

   usage: http://target_machine:2950/kickuser?username=User

   /listdisks - get info about disks that attached to target machine

   example (usage): curl http://192.168.5.5:2950/listdisks --ntlm --negotiate -u user:pass

   return example: {"State":"success","Message":"Disks info extracted","AdditionalInformation":"[{\"DiskLabel\":\"\",\"DiskName\":\"C:\\\\\",\"FreeSpace\":7703867392,\"TotalSize\":125707481088,\"DriveType\":3,\"Ready\":true},{\"DiskLabel\":\"Новий том\",\"DiskName\":\"D:\\\\\",\"FreeSpace\":48635392000,\"TotalSize\":114225573888,\"DriveType\":3,\"Ready\":true},{\"DiskLabel\":\"Новий том\",\"DiskName\":\"E:\\\\\",\"FreeSpace\":83083874304,\"TotalSize\":4000768323584,\"DriveType\":3,\"Ready\":true}]"}


   /reboot - reboot target machine

   usage: http://192.168.5.5:2950/reboot


   # Permissions
   WinHTTPAPI user auth based on ntlm authorization.
   
   By default, methods:
   /remotecmd, /mkdir, /upload, /copy, /move, /remove, /kickuser

   only can be executed by WinHTTPAPIAdministrators group member, other methods can be executed by both WinHTTPAPIAdministrators and WinHTTPAPIUsers groups members

   Users who do not added to any of that two groups, will be rejected.
