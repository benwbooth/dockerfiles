# docker build --build-arg sshpass= --build-arg vpnuser= --build-arg vpnpass= -t benwbooth/qbittorrent qbittorrent
# docker run -d --name qbittorrent --cap-add=SYS_ADMIN --cap-add=NET_ADMIN --device=/dev/net/tun --security-opt=seccomp:unconfined -v /sys/fs/cgroup:/sys/fs/cgroup:ro -v h:/:/mnt/Downloads -v i:/:/mnt/data -v j:/:/mnt/data2 -v k:/:/mnt/external -p 2222:22 -p 8080:8080 benwbooth/qbittorrent
# docker run -d --name qbittorrent --privileged --device=/dev/net/tun -v /sys/fs/cgroup:/sys/fs/cgroup:ro -v h:/:/mnt/Downloads -v i:/:/mnt/data -v j:/:/mnt/data2 -v k:/:/mnt/external -p 2222:22 -p 8080:8080 benwbooth/qbittorrent
FROM base/archlinux:latest
ENV container docker
ARG vpnuser
ARG vpnpass
ARG sshpass
ARG region=Sweden
ENV REGION=${region}

# install/update packages
RUN pacman --noconfirm -Sy archlinux-keyring
RUN pacman --noconfirm -Sy pacman pacman-mirrorlist \
    && rm -rf /etc/pacman.d/mirrorlist.pacnew \
    && pacman-db-upgrade \
    && pacman -Syu --noconfirm --needed git base base-devel openssh qbittorrent-nox atool cpio lha lzop p7zip unace unrar zip unzip sudo openvpn ufw systemd-sysvcompat wget vim htop tmux openresolv rsync

# fix systemd services
RUN (cd /usr/lib/systemd/system/sysinit.target.wants && for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /usr/lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /usr/lib/systemd/system/local-fs.target.wants/*; \
rm -f /usr/lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /usr/lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /usr/lib/systemd/system/basic.target.wants/*;\
rm -f /usr/lib/systemd/system/anaconda.target.wants/*;

# configure ssh
RUN sed -i \
        -e 's/^#*\(PermitRootLogin\) .*/\1 no/' \
        -e 's/^#*\(PasswordAuthentication\) .*/\1 yes/' \
        -e 's/^#*\(PermitEmptyPasswords\) .*/\1 yes/' \
        -e 's/^#*\(UsePAM\) .*/\1 no/' \
        /etc/ssh/sshd_config

# install openvpn profile with DNS fix script
RUN wget https://www.privateinternetaccess.com/openvpn/openvpn.zip && \
sudo unzip openvpn.zip -d /etc/openvpn/client && \
for f in /etc/openvpn/client/*; do [[ $f =~ [.]crt$ || $f =~ [.]pem$ ]] && continue; sed -i -e 's/^\(auth-user-pass\)/\1 \/etc\/openvpn\/login.conf/' "$f" && echo -e '\n# This updates the resolvconf with dns settings\nsetenv PATH /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\nscript-security 2\nup /etc/openvpn/update-resolv-conf.sh\ndown /etc/openvpn/update-resolv-conf.sh\ndown-pre' >>"$f" && mv -v "$f" "$(f=${f%.ovpn}; echo "${f// /_}").conf"; done && \
echo -e "$vpnuser\n$vpnpass" >/etc/openvpn/login.conf && chmod 600 /etc/openvpn/login.conf

# add local user
RUN useradd -m -s /bin/bash ben && \
usermod -a -G wheel ben && \
echo "ben:$sshpass" |chpasswd && \
chmod u+s /sbin/unix_chkpwd /sbin/su /sbin/sudo /sbin/passwd && \
echo '%wheel ALL=(ALL) NOPASSWD: ALL' >>/etc/sudoers
USER ben
WORKDIR /home/ben

# install trizen
RUN git clone https://aur.archlinux.org/trizen.git && \
(cd trizen && makepkg -si --noconfirm)

# clone dotfiles
RUN git init && \
git remote add origin https://github.com/benwbooth/dotfiles.git && \
git fetch --all && \
git reset --hard origin/master && \
git branch --set-upstream-to=origin/master master

# download scripts, config files and service files
COPY qbittorrent.service qbittorrent-update-port.service ufw4vpn.service openvpn-reconnect.service openvpn-reconnect.timer /etc/systemd/system/
COPY qbittorrent-extract qbittorrent-update-port ufw4vpn openvpn-reconnect /usr/bin/
RUN mkdir -p ~/.config/qBittorrent ~/.local/share/data/qBittorrent
COPY qBittorrent.conf /home/ben/.config/qBittorrent/
RUN sudo chown -R ben:ben ~/.config/qBittorrent ~/.local/share/data/qBittorrent

# set up systemd services
RUN sudo systemctl enable sshd "openvpn-client@$region" qbittorrent qbittorrent-update-port ufw4vpn openvpn-reconnect

# install resolvconf update script
#RUN git clone https://github.com/masterkorp/openvpn-update-resolv-conf.git && \
#sudo mv openvpn-update-resolv-conf/update-resolv-conf.sh /etc/openvpn/
COPY update-resolv-conf.sh /etc/openvpn/

EXPOSE 22 8080
USER root
VOLUME ["/sys/fs/cgroup"]
CMD ["/usr/sbin/init"]