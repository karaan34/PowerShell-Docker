# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License.
#
# Docker image file that describes an Arch Linux image with PowerShell
# installed from full PowerShell linux tar.gz package

# Define arg(s) needed for the From statement
ARG fromTag=latest
ARG imageRepo=archlinux/base

FROM ${imageRepo}:${fromTag} AS installer-env

RUN \
    # create a builduser
    useradd builduser -m \
    # delete the builduser password
    && passwd -d builduser \
    # enable builduser to sudo without password
    && printf 'root ALL=(ALL) ALL\n' | tee -a /etc/sudoers \
    # update package database
    && pacman -Syy \
    # install dependencies
    && pacman -S --noconfirm \
      # required sudo
      sudo \
      # required for uncompressing tar files
      tar \
      # required to clone gss-ntlmssp git repository
      git \
      # required to make gss-ntlmssp binary package
      base-devel \
      libwbclient \
      docbook-xsl \
      doxygen \
    # change current root path to tmp directory path
    && cd /tmp \
    # clone the gss-ntlmssp package repository
    && git clone https://aur.archlinux.org/gss-ntlmssp.git \
    # change the cloned gss-ntlmssp package repository directory permission
    && chmod 777 /tmp/gss-ntlmssp/ \
    # change current path to gss-ntlmssp package repository folder path
    && cd gss-ntlmssp \
    # utilise sudo to builduser in order to make the gss-ntlmssp package
    && sudo -u builduser bash -c makepkg -s --noconfirm

# Start a new stage so we lose all the tar.gz layers from the final image
FROM ${imageRepo}:${fromTag}

# Copy only the files we need from the previous stage
COPY --from=installer-env ["/tmp/gss-ntlmssp/gss-ntlmssp-0.8.0.r3.g2251a72-1-x86_64.pkg.tar.xz", "/tmp/gss-ntlmssp-0.8.0.r3.g2251a72-1-x86_64.pkg.tar.xz"]

# Define Args for the needed to add the package
ARG PS_VERSION=6.2.3
ARG PS_PACKAGE=powershell-${PS_VERSION}-linux-x64.tar.gz
ARG PS_PACKAGE_URL=https://github.com/PowerShell/PowerShell/releases/download/v${PS_VERSION}/${PS_PACKAGE}
ARG PS_INSTALL_VERSION=7

# Define Args and Env needed to create links
ENV PS_INSTALL_FOLDER=/opt/microsoft/powershell/$PS_INSTALL_VERSION \
    \
    # Define ENVs for Localization/Globalization
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    # Set up PowerShell module analysis cache path
    PSModuleAnalysisCachePath=/var/cache/microsoft/powershell/PSModuleAnalysisCache/ModuleAnalysisCache \
    # Opt out of SocketsHttpHandler in DotNet Core 2.1 to use HttpClientHandler
    # with installed libcurl4 package to resolve
    # Invoke-WebRequest : Authentication failed" issue when executing using
    # docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [Invoke-WebRequest] [-Uri <HTTPS URL>]
    DOTNET_SYSTEM_NET_HTTP_USESOCKETSHTTPHANDLER=0 \
    POWERSHELL_DISTRIBUTION_CHANNEL=PSDocker-ArchLinux-${fromTag}

# Installation
RUN \
    # enable en_US.UTF-8 locale
    echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen \
    # generate locale
    && locale-gen \
    # update package database
    && pacman -Syy \
    # install dependencies
    && pacman -S --noconfirm \
      # required for updating Arch Linux keyring
      archlinux-keyring \
      # required for International Components for Unicode
      core/icu \
      # required for SSL
      openssl-1.0 \
      # required for uncompressing tar files
      tar \
    # install gss-ntlmssp package from the previous stage
    && pacman -U --noconfirm /tmp/gss-ntlmssp-0.8.0.r3.g2251a72-1-x86_64.pkg.tar.xz \
    # create powershell folder
    && mkdir -p ${PS_INSTALL_FOLDER} \
    # Pull down the powershell linux tar file
    && curl -sSL ${PS_PACKAGE_URL} -o /tmp/powershell-linux.tar.gz \
    # uncompress powershell linux tar file
    && tar zxf /tmp/powershell-linux.tar.gz -C ${PS_INSTALL_FOLDER} -v \
    # remove powershell linux tar file
    && rm -f /tmp/powershell-linux.tar.gz \
    # Create the pwsh symbolic link that points to powershell
    && ln -s ${PS_INSTALL_FOLDER}/pwsh /usr/bin/pwsh \
    # Give all user execute permissions and remove write permissions for others
    && chmod a+x,o-w ${PS_INSTALL_FOLDER}/pwsh \
    # intialize powershell module cache
    && pwsh \
        -NoLogo \
        -NoProfile \
        -Command " \
          \$ErrorActionPreference = 'Stop' ; \
          \$ProgressPreference = 'SilentlyContinue' ; \
          while(!(Test-Path -Path \$env:PSModuleAnalysisCachePath)) {  \
            Write-Host "'Waiting for $env:PSModuleAnalysisCachePath'" ; \
            Start-Sleep -Seconds 6 ; \
          }" \
    # upgrade distro
    && pacman -Syyu --noconfirm \
    # clean downloaded packages
    && yes | pacman -Scc

# Define args needed only for the labels
ARG IMAGE_NAME=pshorg/powershellcommunity:archlinux-2018.10.01
ARG VCS_REF="none"

# Add label last as it's just metadata and uses a lot of parameters
LABEL maintainer="PowerShell Team <powershellteam@hotmail.com>" \
      readme.md="https://github.com/PowerShell/PowerShell/blob/master/docker/README.md" \
      description="This Dockerfile will install the latest release of PowerShell." \
      org.label-schema.usage="https://github.com/PowerShell/PowerShell/tree/master/docker#run-the-docker-image-you-built" \
      org.label-schema.url="https://github.com/PowerShell/PowerShell/blob/master/docker/README.md" \
      org.label-schema.vcs-url="https://github.com/PowerShell/PowerShell-Docker" \
      org.label-schema.name="powershell" \
      org.label-schema.vendor="PowerShell" \
      org.label-schema.vcs-ref=${VCS_REF} \
      org.label-schema.version=${PS_VERSION} \
      org.label-schema.schema-version="1.0" \
      org.label-schema.docker.cmd="docker run ${IMAGE_NAME} pwsh -c '$psversiontable'" \
      org.label-schema.docker.cmd.devel="docker run ${IMAGE_NAME}" \
      org.label-schema.docker.cmd.test="docker run ${IMAGE_NAME} pwsh -c Invoke-Pester" \
      org.label-schema.docker.cmd.help="docker run ${IMAGE_NAME} pwsh -c Get-Help"

CMD [ "pwsh" ]
