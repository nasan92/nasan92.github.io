---
title: 'How to show last bootuptime of Computers with Powershell'
author: Nathanael Santschi
date: 2021-05-12
tags:
  - Powershell
---

## Use Get-CimInstance with the ClassName win32_operatingsystem
You can query the ClassName win32_operatingsystem and select from there the computername **csname** and the last bootup time **lastbootuptime**
````powershell
Get-CimInstance -ComputerName 'DC01' -ClassName win32_operatingsystem | select csname, lastbootuptime
````
## Show last bootuptime for multiple computers
If you want to do that for multiple computers you can for example do it wit a foreach loop:
````powershell
# First query the computer from which you want to have this information:
$computer = Get-ADComputer -Filter * | where Name -Match dc
# Now get the lastbootuptime for every computer in the computer array:
foreach($c in $computer){
    $computername = $c.DNSHostname
    Get-CimInstance -ComputerName $computername -ClassName win32_operatingsystem | select csname, lastbootuptime
}
````
## Which other information can you get from this class?
If you're curious what other information you can get from this class you can simply run a Get-Member which will list you all properties and methods:
````powershell
Get-CimInstance -ComputerName $computername -ClassName win32_operatingsystem | Get-Member


   TypeName: Microsoft.Management.Infrastructure.CimInstance#root/cimv2/Win32_OperatingSystem

Name                                      MemberType   Definition                                                                                                        
----                                      ----------   ----------                                                                                                        
Clone                                     Method       System.Object ICloneable.Clone()                                                                                  
Dispose                                   Method       void Dispose(), void IDisposable.Dispose()                                                                        
Equals                                    Method       bool Equals(System.Object obj)                                                                                    
GetCimSessionComputerName                 Method       string GetCimSessionComputerName()                                                                                
GetCimSessionInstanceId                   Method       guid GetCimSessionInstanceId()                                                                                    
GetHashCode                               Method       int GetHashCode()                                                                                                 
GetObjectData                             Method       void GetObjectData(System.Runtime.Serialization.SerializationInfo info, System.Runtime.Serialization.StreamingC...
GetType                                   Method       type GetType()                                                                                                    
ToString                                  Method       string ToString()                                                                                                 
PSShowComputerName                        NoteProperty bool PSShowComputerName=True                                                                                      
BootDevice                                Property     string BootDevice {get;}                                                                                          
BuildNumber                               Property     string BuildNumber {get;}                                                                                         
BuildType                                 Property     string BuildType {get;}                                                                                           
Caption                                   Property     string Caption {get;}                                                                                             
CodeSet                                   Property     string CodeSet {get;}                                                                                             
CountryCode                               Property     string CountryCode {get;}                                                                                         
CreationClassName                         Property     string CreationClassName {get;}                                                                                   
CSCreationClassName                       Property     string CSCreationClassName {get;}                                                                                 
CSDVersion                                Property     string CSDVersion {get;}                                                                                          
CSName                                    Property     string CSName {get;}                                                                                              
CurrentTimeZone                           Property     int16 CurrentTimeZone {get;}                                                                                      
DataExecutionPrevention_32BitApplications Property     bool DataExecutionPrevention_32BitApplications {get;}                                                             
DataExecutionPrevention_Available         Property     bool DataExecutionPrevention_Available {get;}                                                                     
DataExecutionPrevention_Drivers           Property     bool DataExecutionPrevention_Drivers {get;}                                                                       
DataExecutionPrevention_SupportPolicy     Property     byte DataExecutionPrevention_SupportPolicy {get;}                                                                 
Debug                                     Property     bool Debug {get;}                                                                                                 
Description                               Property     string Description {get;set;}                                                                                     
Distributed                               Property     bool Distributed {get;}                                                                                           
EncryptionLevel                           Property     uint32 EncryptionLevel {get;}                                                                                     
ForegroundApplicationBoost                Property     byte ForegroundApplicationBoost {get;set;}                                                                        
FreePhysicalMemory                        Property     uint64 FreePhysicalMemory {get;}                                                                                  
FreeSpaceInPagingFiles                    Property     uint64 FreeSpaceInPagingFiles {get;}                                                                              
FreeVirtualMemory                         Property     uint64 FreeVirtualMemory {get;}                                                                                   
InstallDate                               Property     CimInstance#DateTime InstallDate {get;}                                                                           
LargeSystemCache                          Property     uint32 LargeSystemCache {get;}                                                                                    
LastBootUpTime                            Property     CimInstance#DateTime LastBootUpTime {get;}                                                                        
LocalDateTime                             Property     CimInstance#DateTime LocalDateTime {get;}                                                                         
Locale                                    Property     string Locale {get;}                                                                                              
Manufacturer                              Property     string Manufacturer {get;}                                                                                        
MaxNumberOfProcesses                      Property     uint32 MaxNumberOfProcesses {get;}                                                                                
MaxProcessMemorySize                      Property     uint64 MaxProcessMemorySize {get;}                                                                                
MUILanguages                              Property     string[] MUILanguages {get;}                                                                                      
Name                                      Property     string Name {get;}                                                                                                
NumberOfLicensedUsers                     Property     uint32 NumberOfLicensedUsers {get;}                                                                               
NumberOfProcesses                         Property     uint32 NumberOfProcesses {get;}                                                                                   
NumberOfUsers                             Property     uint32 NumberOfUsers {get;}                                                                                       
OperatingSystemSKU                        Property     uint32 OperatingSystemSKU {get;}                                                                                  
Organization                              Property     string Organization {get;}                                                                                        
OSArchitecture                            Property     string OSArchitecture {get;}                                                                                      
OSLanguage                                Property     uint32 OSLanguage {get;}                                                                                          
OSProductSuite                            Property     uint32 OSProductSuite {get;}                                                                                      
OSType                                    Property     uint16 OSType {get;}                                                                                              
OtherTypeDescription                      Property     string OtherTypeDescription {get;}                                                                                
PAEEnabled                                Property     bool PAEEnabled {get;}                                                                                            
PlusProductID                             Property     string PlusProductID {get;}                                                                                       
PlusVersionNumber                         Property     string PlusVersionNumber {get;}                                                                                   
PortableOperatingSystem                   Property     bool PortableOperatingSystem {get;}                                                                               
Primary                                   Property     bool Primary {get;}                                                                                               
ProductType                               Property     uint32 ProductType {get;}                                                                                         
PSComputerName                            Property     string PSComputerName {get;}                                                                                      
RegisteredUser                            Property     string RegisteredUser {get;}                                                                                      
SerialNumber                              Property     string SerialNumber {get;}                                                                                        
ServicePackMajorVersion                   Property     uint16 ServicePackMajorVersion {get;}                                                                             
ServicePackMinorVersion                   Property     uint16 ServicePackMinorVersion {get;}                                                                             
SizeStoredInPagingFiles                   Property     uint64 SizeStoredInPagingFiles {get;}                                                                             
Status                                    Property     string Status {get;}                                                                                              
SuiteMask                                 Property     uint32 SuiteMask {get;}                                                                                           
SystemDevice                              Property     string SystemDevice {get;}                                                                                        
SystemDirectory                           Property     string SystemDirectory {get;}                                                                                     
SystemDrive                               Property     string SystemDrive {get;}                                                                                         
TotalSwapSpaceSize                        Property     uint64 TotalSwapSpaceSize {get;}                                                                                  
TotalVirtualMemorySize                    Property     uint64 TotalVirtualMemorySize {get;}                                                                              
TotalVisibleMemorySize                    Property     uint64 TotalVisibleMemorySize {get;}                                                                              
Version                                   Property     string Version {get;}                                                                                             
WindowsDirectory                          Property     string WindowsDirectory {get;}                                                                                    
FREE                                      PropertySet  FREE {FreePhysicalMemory, FreeSpaceInPagingFiles, FreeVirtualMemory, Name}                                        
PSStatus                                  PropertySet  PSStatus {Status, Name}                                                                                           
````

