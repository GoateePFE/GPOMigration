# GPOMigration
PowerShell module and sample code for migrating group policies between domains or forests

Find the blog posts describing this code here: [https://blogs.technet.microsoft.com/ashleymcglone/tag/gpo/](https://blogs.technet.microsoft.com/ashleymcglone/tag/gpo/)

# The Problem
Have you ever wanted to copy all of your production Group Policy Objects (GPOs) into a lab for testing?  Do you have to copy GPOs between domains or forests?  Do you need to migrate them to another environment due to an acquisition, merger, or divestiture? These are common problems for many administrators.

There are VBScripts provided with the Group Policy Management Console (GPMC), but that is so "last decade". (Really. They were published in 2002.)  What about WMI filters, OU links, login scripts, and embedded credentials? I’ve drafted a PowerShell module to do this with speed and style. This post discusses the pitfalls, preparations, and scripts for a successful GPO migration.

# Real-World Scenario
Recently I worked with a customer who had mirrored dev, test, and prod Active Directory forests.  They had the same accounts, groups, OUs, and GPOs in all three places.  Then they had another version of the same dev, test, prod environment for a separate application.  That is two sets of three forests, both with identical GPOs.  Their current process for copying policies was manually backing up and importing the GPOs, which is how TechNet tells you to do it.  At this scale, however, they were in need of an automated solution.  Enter PowerShell.

# Scripting Options
When automating Group Policy with the tools in the box you have three options:

1. Group Policy Management Console (GPMC) VBScripts (circa 2002)
2. GroupPolicy PowerShell module (Windows Server 2008 R2 and above, installed with GPMC)
3. GPMgmt.GPM COM object which is the secret sauce behind #1 and #2

VBScript.  Yeah.  That worked great all those years ago.  I know.  That’s what I used day-in-day-out before PowerShell.  But this is a new era.  If you are still relying on VBScript, then it is time for an intervention from your peers.

My default choice is always to use the cmdlets out-of-the-box.  And that it what I tried to do for the most part.  However, while developing this solution I ran into a number of limitations with the GroupPolicy module cmdlets.  I’ll detail those below.

Behind the VBScripts and the cmdlets there is a COM object called “GPMgmt.GPM”.  Here is a list of the methods exposed by the object:
```PowerShell
PS C:\> New-Object -ComObject GPMgmt.GPM | Get-Member | Select-Object Name
```
```text
Name 
---- 
CreateMigrationTable 
CreatePermission 
CreateSearchCriteria 
CreateTrustee 
GetBackupDir 
GetBackupDirEx 
GetClientSideExtensions 
GetConstants 
GetDomain 
GetMigrationTable 
GetRSOP 
GetSitesContainer 
InitializeReporting 
InitializeReportingEx
```

For example, the Get-GPResultantSetOfPolicy cmdlets calls the GetRSOP method of this COM object.  However, we do not have full cmdlet coverage.  There are no cmdlets for working with GPO migration tables.  Therefore I studied the migration table VBScripts and essentially converted them to PowerShell.  The VBScripts have great value as templates for how to use this COM object.  It’s just not cool to rely on VBScript for much else these days.

# GPO Scripting Challenges
When I first sat down to tackle GPO migration I found the convenient cmdlet Copy-GPO.  Game over, right?  Just use the cmdlet.  Oh, how I wish it were that easy.  To make a very long story very short here is a summary of the challenges I encountered:

* Copy-GPO requires both source and destination domains to be online.  That means we cannot use it for disconnected dev, test, prod forest scenarios.  No problem.  I’ll just use Backup-GPO and Import-GPO…
* Backup-GPO/Import-GPO does not have the -CopyACL switch from Copy-GPO.  Now I have to find another way to migrate permissions.  No problem.  I’ll just use the Get-GPPermission/Set-GPPermission cmdlets…
* Set-GPPermission will not set deny entries, only allow.  Seriously?  Some shops rely on deny.  I had to write my own code for this piece, and it was quite involved.  However, I used the opportunity to translate permissions based on the migration table, so that made it more robust in the end.
* As mentioned above there are no cmdlets for Group Policy Migration Tables.  This is a necessary evil for most GPO migrations.  Restricted groups, user rights assignment, script paths, etc. can be buried down in the policies.  Migration tables tell the import how to translate accounts and paths in policies to the new domain.  Usually creating a migration table is a manual process with an ancient GUI tool.  I automated the whole thing using a simple CSV file where you can specify search/replace values to automatically update the automatically generated migration table.
* Import-GPO has a switch to use a migration table, but it forces the option from the GUI which requires all accounts to be in the migration table.  I left this one as-is.  You can work around this by adjusting the migration table or fudging accounts.
* Neither Copy-GPO nor Import-GPO support WMI filter migration.  After extensive research I discovered that WMI filter scripting may require a registry hack and a DC reboot due to a “system owned object” feature.  This one is the ugliest of them all, and I decided to leave it alone.  Bin Yi from Microsoft has posted a PowerShell module on the TechNet Script Gallery for migrating WMI filters.  Feel free to use his code if you need this functionality.  Backup-GPO puts all the WMI filter data into the backup, but writing it back to the new environment is the challenge.  I’ll tackle this later if I have demand for it.
In this case the old saying is true, “It is never as easy as it looks.”

# The Process
If there ever were a case for automation this is it.  The export process allows us to do multiple GPOs simultaneously, and some of the import steps are optional.  Even so, it is quite involved.  Here is the complete, manual GPO migration process:

1. Export GPOs from source domain
2. Copy export files to destination domain
3. Create and tweak migration table
4. Manually recreate WMI filters in destination
5. Remove GPOs of same name in destination
6. Import GPOs to destination domain
7. Manually reassign WMI filters
8. Copy permissions (and sync SYSVOL permissions)
9. Link GPOs to OUs
10. Set link properties (enabled, enforced, etc.)
Now imagine repeating that effort… multiple times... by hand… without making any mistakes… without forgetting a step… and keeping your sanity.

Beginner Tip:  If you have never done a GPO backup and import from the GUI, then I suggest you start there first.  That will give you a better idea of the overall process.  You will want to click the option for the migration table so that you understand it as well.

# The Solution
My mission is to make things simple for Microsoft customers.  I was able to reduce the entire manual process down to a new PowerShell module and a CSV file.  Here is an outline of the new module cmdlets involved.  You will notice these correlate directly to the process steps above (except for WMI not supported in this release).

* Start-GPOExport
  * Invoke-BackupGPO
    * (Backup-GPO)
    * Export-GPPermission
* Start-GPOImport
  * New-GPOMigrationTable
  * Show-GPOMigrationTable
  * Test-GPOMigrationTable
  * Invoke-RemoveGPO
    * (Remove-GPO)
  * Invoke-ImportGPO
    * (Import-GPO)
    * Import-GPPermission
  * Import-GPLink
 

Let's break this down into three steps, well four if you count the setup, or maybe five if you count extra tinkering.

# Step 0 – Setup
In the source domain and destination domain you want a workstation or member server with the following basic requirements:

* PowerShell version 2 or above
* Remote Server Administration Tools (RSAT)
  * Active Directory module
  * Group Policy module
  * GPMC
On your machine set up a working folder where you copy the PowerShell files from this blog post.  The download link is at the bottom of the article.  By the way, you will usually need to unblock the file(s) after download.

I developed this on a Windows 8.1 client running PowerShell v4 and tested it on Windows Server 2008 R2 (PSv2), Windows Server 2012 (PSv3), and Windows Server 2012 R2 (PSv4).

# Step 1 – Migration Table CSV File
We will call this the “migration table CSV file”.  It is not a GPO  migration table, but it feeds the automation process behind building and updating the migration table.  Before we run the migration code we need to create a simple CSV file that maps source domain references to the destination domain.  Here is an example that is included with the code:
```text
Source               Destination         Type
------               -----------         ----
wingtiptoys.local    cohovineyard.com    Domain
wingtiptoys          cohovineyard        Domain
\\wingtiptoys.local\ \\cohovineyard.com\ UNC
\\wingtiptoys\       \\cohovineyard\     UNC
```
Notice there are short name (NetBIOS) and long name (FQDN) entries for each domain and for both “Domain” and “UNC” type.  You can add other values for server names in UNC paths, etc.  This is my suggested minimum.  You will want one of these files for each combination of source/destination domains where you are migrating GPOs.  Make copies of the sample and modify them to your needs.

# Step 2 – Export
The ZIP download includes a sample calling script for the export.  All you have to do is update the working folder path, modify the domain and server names, and then edit the Where-Object line to query the GPO(s) you want to migrate.
```PowerShell
Set-Location "C:\Temp\GPOMigration\"            
            
Import-Module GroupPolicy            
Import-Module ActiveDirectory            
Import-Module ".\GPOMigration.psm1" -Force            
            
# This path must be absolute, not relative            
$Path        = $PWD  # Current folder specified in Set-Location above            
$SrceDomain  = 'wingtiptoys.local'            
$SrceServer  = 'dca.wingtiptoys.local'            
$DisplayName = Get-GPO -All -Domain $SrceDomain -Server $SrceServer |            
    Where-Object {$_.DisplayName -like '*test*'} |             
    Select-Object -ExpandProperty DisplayName            
            
Start-GPOExport `
    -SrceDomain $SrceDomain `
    -SrceServer $SrceServer `
    -DisplayName $DisplayName `
    -Path $Path            
```
Run the script.  This calls the necessary module functions to create the GPO backup and export the permissions.  Note that the permissions are listed in the GPO backup, but there is no practical way to decipher them.  (Trust me.  Long story.)  In this case we’re going to dump the permissions to a simple CSV that gets written into the same GPO backup folder.

The working folder will now include a subfolder with the GPO backup.  Copy the entire working folder to your destination domain working machine.

# Step 3 – Import
This is where most of the fancy foot work takes place, but I’ve reduced it to “one big button” if that meets your needs.  The ZIP download includes a sample calling script for the import.  This time you have to update the working folder path, modify the domain and server names, update the backup folder path, and then update the migration table CSV path to point to the file you created in Step 1 above.

Note:  Be sure not to confuse the source and destination domain/server names.  It would be unfortunate if you got those backwards when working in a production environment.  Just sayin’.  You’ve been warned.
```PowerShell
Set-Location "C:\Temp\GPOMigration\"            
            
Import-Module GroupPolicy            
Import-Module ActiveDirectory            
Import-Module ".\GPOMigration.psm1" -Force            
            
# This path must be absolute, not relative            
$Path        = $PWD  # Current folder specified in Set-Location above            
$BackupPath  = "$PWD\GPO Backup wingtiptoys.local 2014-04-23-16-37-31"            
$DestDomain  = 'cohovineyard.com'            
$DestServer  = 'cvdcr2.cohovineyard.com'            
$MigTableCSVPath = '.\MigTable_sample.csv'            
            
Start-GPOImport `
    -DestDomain $DestDomain `
    -DestServer $DestServer `
    -Path $Path `
    -BackupPath $BackupPath `
    -MigTableCSVPath $MigTableCSVPath `
    -CopyACL            
```
Run the script.  This calls the necessary module functions to import each GPO from the backup and put everything back in place in the destination domain.  After the script finishes review the output.  Check for any errors.  Verify the results in the destination domain using GPMC.  You can always rerun the script as many times as you like, making adjustments each time.

The working folder will now include a *.migtable file for the GPO migration table.  You can view and edit this, but be aware that the default logic in Start-GPOImport will create a new one each time.  Using Start-GPOImport requires to have the same accounts in the source and destination domains.  You can adjust the migration table and instead use Invoke-ImportGPO directly with your custom migration table.  Most likely the migration table will take some time to smooth out.  You’ll catch on.

Also be aware that by default Start-GPOImport removes any existing GPOs with the same name.  This is by design.  Remember that you can tweak the Start-GPOImport function to suit your own needs.

# Step 4 – Free Style
Once you get the hang of the process I encourage you to dive into the Start-GPOImport function contained in the module.  It is pre-set to do a full import.  Your needs will likely vary from this template.  Use the syntax from this function to build your own import routine tailored to your requirements.

# Summary
In a nut shell I’ve taken a multiple step manual process and condensed it down to three simple steps that execute quickly in PowerShell.  I agree that it is a pain to update paths in the calling script and copy files around.  On the bright side it is still way faster than the manual alternative.

As always when you are copying scripts from the internet make sure that you understand what the script will do before you run it.  Test it in a lab before using it in production.  Open up the GPOMigration.psm1 module file and skim through the code.  Review the full help content for each function.  You will learn more PowerShell and get ideas for your own scripts.

I’d love to hear how this script module has helped you.  Please use the comments below to ask questions and offer feedback.  Put your best foot forward with PowerShell!

 

# Part 2 - WMI Filters
 

I received feedback that WMI filters must be supported before this would be considered a viable solution. So I went back to my lab, integrated some code from the TechNet Script Center, and we have version 1.1 now, including WMI filter migration.  Bin Yi, the author of the WMIFilter module on the TechNet Script Center, graciously permitted me to borrow some of his code to make the magic happen.

 

# Added Functionality for WMI Filters
 

As discussed in my previous article on this topic we noted that WMI filters are a key part of the migration process. The WMI filter information is included in the GPO backup data, but it is not easy to retrieve.  Therefore I wrote a simple routine to dump the pertinent WMI filter attributes into a CSV file in the GPO backup folder. We only export and import WMI filters pertinent to the GPO migration. In other words, we do not blindly migrate all WMI filters from the source environment. If the filter is already there, then we do not recreate it. If the filter is indeed missing, then we create it in the destination.

 

This new functionality is covered in three function:

 

* Export-WMIFilter – Creates a CSV file of WMI filters from the source environment. The Invoke-BackupGPO function supplies the list of WMI filters to include in the export.
* Import-WMIFilter – Reads the WMI CSV file and creates the filters in the destination environment.
* Set-GPWMIFilterFromBackup – Reads the GPO backup information and links WMI filters to the corresponding GPOs of the import.
* Enable-ADSystemOnlyChange – Sets a registry flag on the domain controller, allowing it to create the WMI filter objects.
 

Here is the updated map of functions in this module:

* Start-GPOExport
  * Invoke-BackupGPO
    * (Backup-GPO)
    * Export-WMIFilter
  * Export-GPPermission
* Enable-ADSystemOnlyChange (optional)
* Start-GPOImport
  * New-GPOMigrationTable
  * Show-GPOMigrationTable
  * Test-GPOMigrationTable
  * Invoke-RemoveGPO
    * (Remove-GPO)
  * Invoke-ImportGPO
    * (Import-GPO)
    * Import-GPPermission
  * Import-WMIFilter
  * Set-GPWMIFilterFromBackup
  * Import-GPLink
 

This improves our previous process by removing the manual WMI steps.

 

# WMI Filter Active Director Objects
 

Let’s take a journey down a side road into your AD database and find these WMI filters. They live in this path:

 

CN=SOM,CN=WMIPolicy,CN=System,DC=contoso,DC=com
 

If you turn on the Advanced Features view in Active Directory Users & Computers (ADUC) you can drill down and find this as pictured below:

 

image

 

Here is a view of the properties on these WMI objects:

 

image

 

We are primarily interested in the following properties and what they contain:

 

* msWMI-Author – not truly needed, but good information to retain
* msWMI-Name – display name
* msWMI-Parm1 – description
* msWMI-Parm2 – WQL and some other jazz
 

These are the four properties we need to migrate for each WMI filter. We export these to a CSV file. Notice that the actual Name property is the GUID. The DisplayName is stored in msWMI-Name.

 

# There is always a catch.
 

Now that we have the property values you would think it is as easy as New-ADObject. Um, no. There are a couple challenges to creating WMIFilter objects:

 

1. You have to generate the data values for the other attributes (CreationDate, ChangeDate, GUID, etc.). Those are interesting but manageable.
2. The real issue is AD System Only Change.
 

This post over on the AskDS blog explains that before you can create a WMI filter object you have to stand on your left leg, jump three times, and then throw salt over your right shoulder.  Well, not exactly.  That would be easier. For older operating systems you may have to add a registry key and reboot the DC prior to creating WMI filters. This would also mean that you carefully target said enabled DC with the object create cmdlets. This was not an issue on my Windows Server 2008 R2 or newer DCs in the lab.

 

In case this is an issue for you, I modified some of Bin Yi’s code and put it into this function: Enable-ADSystemOnlyChange. It modifies the domain controller registry to enable/disable WMI object creation and then does a restart (which prompts you first). Here is the registry info for reference:


* HKLM:\System\CurrentControlSet\Services\NTDS\Parameters
  * "Allow System Only Change", DWORD, 1 or 0
 

Fun stuff.

 

# Module Housekeeping
 

The previous release of this code was a simple PSM1 script module file. Since then I’ve been learning alternate ways to manage PowerShell help. In this release I created a module manifest and moved the help from inline comments to an external XML file. I would like to thank Vadims for his project on CodePlex that helps generate these help XML files.

 

To use the code in the download now you will extract the GPOMigration folder to your hard drive. Then type something like this:
```PowerShell
PS> cd (to folder containing the module folder)
PS> Import-Module .\GPOMigration
PS> Get-Command -Module GPOMigration
PS> Get-Help Export-WMIFilter -Full
PS> etc....
```

You will also get some sample files for calling the export and import routines. See the first post for more information on these.

 

# Conclusion
 

Now you know why I tabled this feature for a later release. The good news is it is done. Now you have a GPO migration module for PowerShell that also moves WMI filters. Go grab a fresh soda from the machine and let’s start migrating policies.  Yee haw!
