

//192.168.73.128
ssh root@192.168.1.12 -p 6003   

//192.168.73.129
ssh root@192.168.1.12 -p 6002

//192.168.73.130
ssh root@192.168.1.12 -p 6001

sudo yum update

sudo yum install openssh-server

sudo systemctl start sshd

sudo systemctl enable sshd

sudo systemctl status sshd
