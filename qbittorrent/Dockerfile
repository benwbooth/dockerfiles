# docker build --build-arg 'sshpass=password' qbittorrent
# docker run -d --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro -p 2222:22 <image>
FROM base/archlinux:latest
ENV container docker
ARG sshpass=password

# Update Arch
RUN pacman --noconfirm -Sy archlinux-keyring
RUN pacman --noconfirm -Sy pacman pacman-mirrorlist \
    && rm -rf /etc/pacman.d/mirrorlist.pacnew \
    && pacman-db-upgrade \
    && pacman -Syu --noconfirm --needed git base base-devel openssh qbittorrent-nox atool cpio lha lzop p7zip unace unrar zip unzip sudo networkmanager networkmanager-openvpn ufw systemd-sysvcompat wget vim htop tmux

# Clean services
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

# set up systemd services
RUN systemctl enable sshd NetworkManager qbittorrent-nox@ben

# add local user
RUN useradd -m -s /bin/bash ben && \
usermod -a -G wheel ben && \
echo "ben:$sshpass" |chpasswd && \
chmod u+s /sbin/unix_chkpwd /sbin/su /sbin/sudo /sbin/passwd && \
echo '%wheel ALL=(ALL) NOPASSWD: ALL' >>/etc/sudoers
USER ben
WORKDIR /home/ben

# download dotfiles from git
RUN git init && \
git remote add origin https://github.com/benwbooth/dotfiles.git && \
git fetch --all && \
git reset --hard origin/master && \
git branch --set-upstream-to=origin/master master

# install trizen
RUN git clone https://aur.archlinux.org/trizen.git && \
(cd trizen && makepkg -si --noconfirm)

# install openvpn profile and connect
RUN wget https://www.privateinternetaccess.com/openvpn/openvpn.zip && \
unzip openvpn.zip

EXPOSE 22
USER root
VOLUME ["/sys/fs/cgroup"]
CMD ["/usr/sbin/init"]