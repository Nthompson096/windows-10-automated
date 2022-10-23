# TL:DR
### automated windows 11/10 install

**_Note UEFI appaers to not work with virtualiztion so you'd have to set things up manually._**

before we can do that install the ADK and only have windows deployment tools checked during the install then proceed with the ADK install also be sure that you've downloaded the windows 10/11 iso, either extract the ISO with the usb creation tool kid (will require some ESD wrangling) or just get the ISO

## Generating an answer file:

[Use this site to generate these files](https://www.windowsafg.com/); you can edit the file with notepad or ADK (What I'd prefer) and edit or add things out/in.

## I got the ISO from the USB creation toolkit:

ok so now you'll need to enter the following as an admin in cmd

    dism /Get-WimInfo /Wimfile:E:\sources\install.esd
    
    
    dism /Export-Image /SourceImageFile:X:path\to\install\esd/SourceIndex:1 /DestinationImageFile:X:path\to\destination\.wim /Compress:Max /CheckIntegrity

# LET'S GET STARTED

Component names start with amd64_Microsoft-Windows or x86 for 32 bit

add region and language settings to answer file, right click component 
`International-Core-WinPE`, select `Add Setting to Pass 1 windowsPE` and edit the following:

InputLocale: Your preferred default keyboard layout
SystemLocale: Your country or region
UILanguage: Windows language
UserLocale: PC location

to check the regional settings enter `dism /online /get-intl` in an admin cmd prompt

Expand component Microsoft-Windows-International-Core-WinPE__neutral, select SetupUILanguage; enter the lang.

Expand `Microsoft-Windows-International-Core-WinPE__neutral`, select `UserData`, set 
`AcceptEula` to `true`

Expand UserData, select ProductKey, add a generic product key:

    Windows 10 Home Single Language: 7HNRX-D7KGG-3K4RQ-4WPJ4-YTDFH
    Windows 10 Home: TX9XD-98N7V-6WMQ6-BX7FG-H8Q99
    Windows 10 Pro: VK7JG-NPHTM-C97JM-9MPGT-3V66T
    
[More key's here](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys)

Right click Setup > DiskConfiguration in Answer File pane, select Insert New Disk:

Select the disk you added in Answer File pane, set `DiskID` to be `0` and `WillWipeDisk` to `true`

Expand disk in Answer File pane, right click Create Partitions and select 
`Insert New CreatePartition` to create first partition repeat this step and create one more partition. UEFI / GPT based machines requiring at least four partitions; ***this will be discussed below.***

## MBR

Select a partition (CreatePartition) in Answer File pane, `set Order 1`, `Size 450`, `Type Primary`.

    System Reserved False   1   450 Primary
    Windows True    2   Leave empty Primary

right click `ModifyPartitions` and select `Insert New ModifyPartition`
Repeat this to create a `ModifyPartition` setting for each partition you've created.

    Active = True
    Format = NTFS
    Label = System
    Order = 1
    PartitionID = 1

    ModifyPartition 2 (Windows):
    Format = NTFS
    Label = Windows
    Letter = C
    Order = 2
    PartitionID = 2

Expand `ImageInstall` > `OSImage` component in Answer File pane, select `InstallTo`, set `DiskID = 0` and `PartitionID = 2` to tell Windows setup to install Windows on `partition 2`

## GPT

Select a partition (CreatePartition) in Answer File pane, set Order 1, Size 450, Type Primary.

    WinRE   False   1   450 Primary
    EFI False   2   100 EFI
    MSR False   3   16  MSR
    Windows True    4   Leave empty Primary

right click `ModifyPartitions` and select `Insert New ModifyPartition`
Repeat this to create a `ModifyPartition` setting for each partition you've created.
    
    Format = NTFS
    Label = WinRE
    Order = 1
    PartitionID = 1
    TypeID = DE94BBA4-06D1-4D40-A16A-BFD50179D6AC
    
    ModifyPartition 2 (EFI):
    Format = FAT32
    Label = System
    Order = 2
    PartitionID = 2
    
    ModifyPartition 3 (MSR):
    Order = 3
    PartitionID = 3
    
    ModifyPartition 4 (Windows):
    Format = NTFS
    Label = Windows
    Letter = C
    Order = 4
    PartitionID = 4

expand `ImageInstall` > `OSImage` component in Answer File pane, select `InstallTo`, set 
`DiskID = 0` and `PartitionID = 4` to tell Windows setup to install Windows on 
`partition 4`

## OOBE

add component Microsoft-Windows-LUA-settings_neutral to pass 2 and set it to `false` 

Add Microsoft-Windows-Security-SPP_Neutral and set skip realm (realm is old school for domains) to `1`

Add these to specialize

amd64_Microsoft-Windows-International-Core__neutral_31bf3856ad364e35_nonSxS

- Set locales and UI lang

amd64_Microsoft-Windows-Security-SPP-UX__neutral_31bf3856ad364e35_nonSxS

- Skip auto activation to true

amd64_Microsoft-Windows-Shell-Setup__neutral_31bf3856ad364e35_nonSxS

- Set the computer name and enter the product key

amd64_Microsoft-Windows-SQMApi__neutral_31bf3856ad364e35_nonSxS

- CEIPEnabled set to `0 `

add these to oobe system

amd64_Microsoft-Windows-Shell-Setup__neutral_31bf3856ad364e35_nonSxS

- registered owner to the account such as the username, would be the same username as in user data in pass 1 though out

autologin and password

- enabled set to `true`

- password `UABhAHMAcwB3AG8AcgBkAA==`

for oobe

- Hide EULA page set to true

- Hide OEM set to true

- Hide OnlineAccount set to true

- Hide Wireless set to true

ProtectYourPC (read more) value can be 1, 2 or 3:

    1 = Recommended (default) level of protection
    2 = Only updates are installed.
    3 = Automatic protection is disabled.


- SkipMachineOOBE set to true
- SkipUserOOBE set to true

- add user accounts > local accounts and insert an new LocalAccount

- set the names of these accounts to the ones you had set for oobe, group is Administrators. would be the same username as in user data in pass 1 though out

- Password set to `UABhAHMAcwB3AG8AcgBkAA==`

You can delete an unused component by selecting it and pressing DEL or right clicking it and selecting Delete

Validate answer file to check for possible errors (Tools > Validate Answer File); you may see deprecation warnings, but no errors; if so then you should be fine.

Save answer file as autounattend.xml wherever you had opened/saved; you may burn it inside the ISO or place it inside a USB.
