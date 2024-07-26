# Vim

Edit column 
  * `Ctrl`+`v` - mark across the column
      * `Shift` 
        * +`a` - append
        * +`i` - insert
      * `r` - replace
      * `d` - delete
      * `c` - change

Configuration (~/.config/nvim/init.vim)
```commandline
set nu
set tabstop=4
set shiftwidth=4
set expandtab
set colorcolumn=80
```

# RegEx
...

# find
```bash
find . -name '*.jpg'
find . -name '*_tests.py' -o -name '*_test.py'
```
# networking
```bash
nc -l <ip> <port>
nc -vz {host} {port}

sudo ufw status
sudo ufw allow <port>
sudo ufw disable <port>
sudo ufw disable
sudo ufw enable
sudo ufw reload

sudo iptables -t nat -v -L -n --line-number
sudo iptables -t nat -D POSTROUTING {number-here}

netstat -tuplen
```
# grep
...
# awk
...
