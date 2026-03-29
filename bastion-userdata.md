#!/bin/bash

dnf update -y
dnf install -y mysql git tmux ansible curl wget vim

systemctl enable sshd
systemctl start sshd
