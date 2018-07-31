# Instructions

## Create self-signed certs

Follow the steps below to create self-signed certificates to use for this template. Note that you will get warnings in your browser when using these certificates as they are unable to be validated, but this will demonstrate the capabilities of using end-to-end SSL on Application Gateway.

Run the following PowerShell commands to create the self-signed certificates. Replace with the appropriate paths, DNS names and passwords as necessary.

```powershell
## ELEVATED POWERSHELL SESION
$BASE_DIR = Get-Location

function Create-CustomCertificate([string]$Site, [string]$Domain, [string]$Password) {
  $BASE_DIR = Get-Location

  $cert = Get-ChildItem -Path $(New-SelfSignedCertificate -dnsname "$SITE.$DOMAIN").pspath
  Export-PfxCertificate -Cert $cert -FilePath "certs/$SITE.pfx" -Password $(ConvertTo-SecureString -String $PASSWORD -Force -AsPlainText)
  Export-Certificate -Cert $cert -FilePath "certs/$SITE.cer"
  [System.Convert]::ToBase64String([System.IO.File]::ReadAllBytes("$BASE_DIR/certs/$SITE.pfx")) > "certs/$SITE.txt"
  [System.Convert]::ToBase64String([System.IO.File]::ReadAllBytes("$BASE_DIR/certs/$SITE.cer")) > "certs/$SITE-cer.txt"
}

# Create Web App 1 Cert
Create-CustomCertificate "web1" "cloudcodeit.com" "AzurePassword@123"

# Create Web App 1 Cert
Create-CustomCertificate "web2" "cloudcodeit.com" "AzurePassword@123"

# Create App Gateway 1 Cert
Create-CustomCertificate "site1" "cloudcodeit.com" "AzurePassword@123"

# Create App Gateway 1 Cert
Create-CustomCertificate "site2" "cloudcodeit.com" "AzurePassword@123"
```

## Deploy Arm Template

Follow the steps below to create deploy this template. Note that you will need a Service Principal ID, but can use your User ObjectId.

```powershell
# Get UserId for KeyVault Service Principal
Connect-AzureAD
$User = Get-AzureADUser -ObjectId "daniel@cloudcodeit.com"

# Create Resource Group
$ResourceGroupName = "appgw-demo"
$Location = "eastus2"
New-AzureRmResourceGroup -Name $ResourceGroupName -Location $Location

# Deploy Template
$BASE_DIR = Get-Location
$Prefix="daniel"  # UNIQUE INDICATOR
$Site1Cert = [IO.File]::ReadAllText("$BASE_DIR\certs\site1.txt")
$Site2Cert = [IO.File]::ReadAllText("$BASE_DIR\certs\site2.txt")
$Site1Auth = [IO.File]::ReadAllText("$BASE_DIR\certs\web1-cer.txt")
$Site2Auth = [IO.File]::ReadAllText("$BASE_DIR\certs\web2-cer.txt")

New-AzureRmResourceGroupDeployment -Name TemplateDemo `
  -TemplateFile azuredeploy.json `
  -prefix $Prefix `
  -servicePrincipalAppId $User.ObjectId  `
  -site1CertData $Site1Cert -site2CertData $Site2Cert `
  -site1AuthData $Site1Auth -site2AuthData $Site2Auth `
  -ResourceGroupName $ResourceGroupName
 ```

## Setup the Web Sites

Follow the steps below to create dummy web sites.

1. Go to the Web Site App Service Editor and add the following files.
  - keepalive.html  # Required for health probe
  - index.html      # Simple Home page

2. Add DNS Records in the domain for the following
   - site1.cloudcodeit.com  # A Record to the App Gateway
   - site2.cloudcodeit.com  # A Record to the App Gateway
   - web1.cloudcodeit.com   # CName Record to the Web App
   - web2.cloudcodeit.com   # CName Record to the Web App

3. Upload the Web Certs and add the SSL Bindings
   - web1.cloudcodeit.com
   - web2.cloudcodeit.com

## Setup the Web Sites
Reference Articles

[ARM API for App Gateways](https://docs.microsoft.com/en-us/azure/templates/Microsoft.Network/applicationGateways?toc=%2Fen-us%2Fazure%2Fazure-resource-manager%2Ftoc.json&bc=%2Fen-us%2Fazure%2Fbread%2Ftoc.json)
[End to End SSL Powershell](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-end-to-end-ssl-powershell)
[SSL Termination](https://docs.microsoft.com/en-us/azure/application-gateway/create-ssl-portal)
[Geo Redundancy](https://www.cameronvetter.com/2018/03/09/using-azure-application-gateway-wafs-to-secure-azure-web-apps-and-traffic-manager-for-geo-redundancy/)
[End to End SSL](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-end-to-end-ssl-powershell)
[302 Redirect](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-redirect-overview)

