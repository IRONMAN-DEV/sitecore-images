# escape=`
FROM mcr.microsoft.com/windows/servercore:1803 as builder

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

COPY . /install/

# Expand zips, prepare SIF config and WDP package
RUN $env:SITENAME = 'sc'; `
    $env:INSTALL_TEMP = 'C:\\install'; `
    $env:SIF_CONFIG = 'sitecore-XM1-cd.json'; `
    $env:SIF_PACKAGE = 'Sitecore*_cd.scwdp.zip'; `
    Expand-Archive -Path (Join-Path $env:INSTALL_TEMP '*.zip') -DestinationPath $env:INSTALL_TEMP; `
    Expand-Archive -Path (Join-Path $env:INSTALL_TEMP '*Configuration files*.zip') -DestinationPath $env:INSTALL_TEMP; `
    $config = Get-Content (Join-Path $env:INSTALL_TEMP $env:SIF_CONFIG) | ConvertFrom-Json; `
    $config.Variables.'Site.PhysicalPath' = 'C:\inetpub\{0}' -f $env:SITENAME; `
    ConvertTo-Json $config -Depth 50 | Set-Content -Path (Join-Path $env:INSTALL_TEMP $env:SIF_CONFIG); `
    Rename-Item -Path (Resolve-Path (Join-Path $env:INSTALL_TEMP $env:SIF_CONFIG)).Path -NewName 'sitecore.json'; `
    Rename-Item -Path (Resolve-Path (Join-Path $env:INSTALL_TEMP $env:SIF_PACKAGE)).Path -NewName 'sitecore.zip';

# Runtime
FROM sitecore-base:9.1.0-windowsservercore-1803

COPY --from=builder ["/install/sitecore.zip", "/install/sitecore.json", "/install/license.xml", "/install/"]
COPY --from=builder ["/install/Sitecore", "/Sitecore"]

# Install Sitecore, cleanup and start site
RUN $env:SITENAME = 'sc'; `
    $env:INSTALL_TEMP = 'C:\\install'; `
    $env:SQL_ADMIN_USER = 'sa'; `
    $env:SQL_ADMIN_PASSWORD = 'HASH-epsom-sunset-cost7!'; `
    $env:IDENTITYSERVER_URL = 'https://127.0.0.1'; `
    Install-SitecoreConfiguration -Path (Join-Path $env:INSTALL_TEMP '\\sitecore.json') -Package (Join-Path $env:INSTALL_TEMP '\\sitecore.zip') `
    -LicenseFile (Join-Path $env:INSTALL_TEMP '\\license.xml') `
    -Sitename $env:SITENAME `
    -SitecoreIdentityAuthority $env:IDENTITYSERVER_URL `
    -SolrUrl "http://solr:8983/solr" `
    -SolrCorePrefix $env:SITENAME `
    -SqlServer "sql" `
    -SqlDbPrefix $env:SITENAME `
    -SqlSecurityUser $env:SQL_ADMIN_USER -SqlSecurityPassword $env:SQL_ADMIN_PASSWORD `
    -SqlWebUser $env:SQL_ADMIN_USER -SqlWebPassword $env:SQL_ADMIN_PASSWORD `
    -SqlFormsUser $env:SQL_ADMIN_USER -SqlFormsPassword $env:SQL_ADMIN_PASSWORD `
    -Skip "CreateHostHeader", "CreateBindings", "StartAppPool", "StartWebsite"; `
    $iisPath = ('IIS:\Sites\{0}' -f $env:SITENAME); `
    Set-WebConfiguration -PSPath $iisPath -Filter '/system.web/customErrors/@mode' -Value 'Off'; `
    Remove-Item -Path 'C:\\*.log'; `
    Remove-Item -Path ('C:\\inetpub\\{0}\\App_Data\\logs' -f $env:SITENAME) -Force -Recurse; `
    Remove-Item -Path $env:INSTALL_TEMP -Force -Recurse; `
    Start-WebAppPool -Name $env:SITENAME; `
    Start-Website -Name $env:SITENAME
