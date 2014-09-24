Title: 你好，世界！
Date: 2014-08-22 16:08
Category: Python
Tags: python
Author: Jason
Summary: 你好，世界！
你好，世界，世界，你好。
```Python

#coding:utf-8

'''

Created on Oct 28, 2012

@author: http://www.yihaomen.com

'''

import socket, threading, os, sys, time  

import hashlib, platform, stat  

      

listen_ip = "192.168.4.128"  

listen_port = 2111  

conn_list = []  

root_dir = "/home/summer/ftp"  

max_connections = 500  

conn_timeout = 120  

      

class FtpConnection(threading.Thread):  

        def __init__(self, fd):  

            threading.Thread.__init__(self)  

            self.fd = fd  

            self.running = True  

            self.setDaemon(True)  

            self.alive_time = time.time()  

            self.option_utf8 = False  

            self.identified = False  

            self.option_pasv = True  

            self.username = ""  

        def process(self, cmd, arg):  

            cmd = cmd.upper();  

            if self.option_utf8:  

                arg = unicode(arg, "utf8").encode(sys.getfilesystemencoding())  

            print "<<", cmd, arg, self.fd  

            # Ftp Command  

            if cmd == "BYE" or cmd == "QUIT":  

                if os.path.exists(root_dir + "/xxftp.goodbye"):  

                    self.message(221, open(root_dir + "/xxftp.goodbye").read())  

                else:  

                    self.message(221, "Bye!")  

                self.running = False  

                return  

            elif cmd == "USER":  

                # Set Anonymous User  

                if arg == "": arg = "anonymous"  

                for c in arg:  

                    if not c.isalpha() and not c.isdigit() and c!="_":  

                        self.message(530, "Incorrect username.")  

                        return  

                self.username = arg  

                self.home_dir = root_dir + "/" + self.username  

                self.curr_dir = "/"  

                self.curr_dir, self.full_path, permission, self.vdir_list,limit_size, is_virtual = self.parse_path("/")  

                if not os.path.isdir(self.home_dir):  

                    self.message(530, "User " + self.username + " not exists.")  

                    return  

                self.pass_path = self.home_dir + "/.xxftp/password"  

                if os.path.isfile(self.pass_path):  

                    self.message(331, "Password required for " + self.username)  

                else:  

                    self.message(230, "Identified!")  

                    self.identified = True  

                return  

            elif cmd == "PASS":  

                if open(self.pass_path).readline().replace('\r','').replace('\n','') == hashlib.md5(arg).hexdigest():  

                    self.message(230, "Identified!")  

                    self.identified = True  

                else:  

                    self.message(530, "Not identified!")  

                    self.identified = False  

                return  

            elif not self.identified:  

                self.message(530, "Please login with USER and PASS.")  

                return  

      

            self.alive_time = time.time()  

            finish = True  

            if cmd == "NOOP":  

                self.message(200, "ok")  

            elif cmd == "TYPE":  

                self.message(200, "ok")  

            elif cmd == "SYST":  

                self.message(200, "UNIX")  

            elif cmd == "EPSV" or cmd == "PASV":  

                self.option_pasv = True  

                try:  

                    self.data_fd = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  

                    self.data_fd.bind((listen_ip, 0))  

                    self.data_fd.listen(1)  

                    ip, port = self.data_fd.getsockname()  

                    if cmd == "EPSV":  

                        self.message(229, "Entering Extended Passive Mode (|||" + str(port) + "|)")  

                    else:  

                        ipnum = socket.inet_aton(ip)  

                        self.message(227, "Entering Passive Mode (%s,%u,%u)." %  

                            (",".join(ip.split(".")), (port>>8&0xff), (port&0xff)))  

                except:  

                    self.message(500, "failed to create data socket.")  

            elif cmd == "EPRT":  

                self.message(500, "implement EPRT later...")  

            elif cmd == "PORT":  

                self.option_pasv = False  

                self.data_fd = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  

                s = arg.split(",")  

                self.data_ip = ".".join(s[:4])  

                self.data_port = int(s[4])*256 + int(s[5])  

                self.message(200, "ok")  

            elif cmd == "PWD" or cmd == "XPWD":  

                if self.curr_dir == "": self.curr_dir = "/"  

                self.message(257, '"''"' + self.curr_dir + '"')  

            elif cmd == "LIST" or cmd == "NLST":  

                if arg != "" and arg[0] == "-": arg = "" # omit parameters  

                remote, local, perm, vdir_list, limit_size, is_virtual = self.parse_path(arg)  

                if not os.path.exists(local):  

                    self.message(550, "failed.")  

                    return  

                if not self.establish(): return  

                self.message(150, "ok")  

                for v in vdir_list:  

                    f = v[0]  

                    if self.option_utf8:  

                        f = unicode(f, sys.getfilesystemencoding()).encode("utf8")  

                    if cmd == "NLST":  

                        info = f + "\r\n"  

                    else:  

                        info = "d%s%s------- %04u %8s %8s %8lu %s %s\r\n" % (  

                            "r" if "read" in perm else "-",  

                            "w" if "write" in perm else "-",  

                            1, "0", "0", 0,  

                            time.strftime("%b %d  %Y", time.localtime(time.time())),  

                            f)  

                    self.data_fd.send(info)  

                for f in os.listdir(local):  

                    if f[0] == ".": continue  

                    path = local + "/" + f  

                    if self.option_utf8:  

                        f = unicode(f, sys.getfilesystemencoding()).encode("utf8")  

                    if cmd == "NLST":  

                        info = f + "\r\n"  

                    else:  

                        st = os.stat(path)  

                        info = "%s%s%s------- %04u %8s %8s %8lu %s %s\r\n" % (  

                            "-" if os.path.isfile(path) else "d",  

                            "r" if "read" in perm else "-",  

                            "w" if "write" in perm else "-",  

                            1, "0", "0", st[stat.ST_SIZE],  

                            time.strftime("%b %d  %Y", time.localtime(st[stat.ST_MTIME])),  

                            f)  

                    self.data_fd.send(info)  

                self.message(226, "Limit size: " + str(limit_size))  

                self.data_fd.close()  

                self.data_fd = 0  

            elif cmd == "REST":  

                self.file_pos = int(arg)  

                self.message(250, "ok")  

            elif cmd == "FEAT":  

                features = "211-Features:\r\nSITES\r\nEPRT\r\nEPSV\r\nMDTM\r\nPASV\r\nREST STREAM\r\nSIZE\r\nUTF8\r\n211 End\r\n"  

                self.fd.send(features)  

            elif cmd == "OPTS":  

                arg = arg.upper()  

                if arg == "UTF8 ON":  

                    self.option_utf8 = True  

                    self.message(200, "ok")  

                elif arg == "UTF8 OFF":  

                    self.option_utf8 = False  

                    self.message(200, "ok")  

                else:  

                    self.message(500, "unrecognized option")  

            elif cmd == "CDUP":  

                finish = False  

                arg = ".."  

            else:  

                finish = False  

            if finish: return  

            # Parse argument ( It's a path )  

            if arg == "":  

                self.message(500, "where's my argument?")  

                return  

            remote, local, permission, vdir_list, limit_size, is_virtual = self.parse_path(arg)  

            # can not do anything to virtual directory  

            if is_virtual: permission = "none"  

            can_read, can_write, can_modify = "read" in permission, "write" in permission, "modify" in permission  

            newpath = local  

            try:  

                if cmd == "CWD":  

                    if(os.path.isdir(newpath)):  

                        self.curr_dir = remote  

                        self.full_path = newpath  

                        self.message(250, '"''"' + remote + '"')  

                    else:  

                        self.message(550, "failed")  

                elif cmd == "MDTM":  

                    if os.path.exists(newpath):  

                        self.message(213, time.strftime("%Y%m%d%I%M%S", time.localtime(  

                            os.path.getmtime(newpath))))  

                    else:  

                        self.message(550, "failed")  

                elif cmd == "SIZE":  

                    self.message(231, os.path.getsize(newpath))  

                elif cmd == "XMKD" or cmd == "MKD":  

                    if not can_modify:  

                        self.message(550, "permission denied.")  

                        return  

                    os.mkdir(newpath)  

                    self.message(250, "ok")  

                elif cmd == "RNFR":  

                    if not can_modify:  

                        self.message(550, "permission denied.")  

                        return  

                    self.temp_path = newpath  

                    self.message(350, "rename from " + remote)  

                elif cmd == "RNTO":  

                    os.rename(self.temp_path, newpath)  

                    self.message(250, "RNTO to " + remote)  

                elif cmd == "XRMD" or cmd == "RMD":  

                    if not can_modify:  

                        self.message(550, "permission denied.")  

                        return  

                    os.rmdir(newpath)  

                    self.message(250, "ok")  

                elif cmd == "DELE":  

                    if not can_modify:  

                        self.message(550, "permission denied.")  

                        return  

                    os.remove(newpath)  

                    self.message(250, "ok")  

                elif cmd == "RETR":  

                    if not os.path.isfile(newpath):  

                        self.message(550, "failed")  

                        return  

                    if not can_read:  

                        self.message(550, "permission denied.")  

                        return  

                    if not self.establish(): return  

                    self.message(150, "ok")  

                    f = open(newpath, "rb")  

                    while self.running:  

                        self.alive_time = time.time()  

                        data = f.read(8192)  

                        if len(data) == 0: break  

                        self.data_fd.send(data)  

                    f.close()  

                    self.data_fd.close()  

                    self.data_fd = 0  

                    self.message(226, "ok")  

                elif cmd == "STOR" or cmd == "APPE":  

                    if not can_write:  

                        self.message(550, "permission denied.")  

                        return  

                    if os.path.exists(newpath) and not can_modify:  

                        self.message(550, "permission denied.")  

                        return  

                    # Check space size remained!  

                    used_size = 0  

                    if limit_size > 0:  

                        used_size = self.get_dir_size(os.path.dirname(newpath))  

                    if not self.establish(): return  

                    self.message(150, "ok")  

                    f = open(newpath, ("ab" if cmd == "APPE" else "wb") )  

                    while self.running:  

                        self.alive_time = time.time()  

                        data = self.data_fd.recv(8192)  

                        if len(data) == 0: break  

                        if limit_size > 0:  

                            used_size = used_size + len(data)  

                            if used_size > limit_size: break  

                        f.write(data)  

                    f.close()  

                    self.data_fd.close()  

                    self.data_fd = 0  

                    if limit_size > 0 and used_size > limit_size:  

                        self.message(550, "Exceeding user space limit: " + str(limit_size) + " bytes")  

                    else:  

                        self.message(226, "ok")  

                else:  

                    self.message(500, cmd + " not implemented")  

            except:  

                self.message(550, "failed.")  

      

        def establish(self):  

            if self.data_fd == 0:  

                self.message(500, "no data connection")  

                return False  

            if self.option_pasv:  

                fd = self.data_fd.accept()[0]  

                self.data_fd.close()  

                self.data_fd = fd  

            else:  

                try:  

                    self.data_fd.connect((self.data_ip, self.data_port))  

                except:  

                    self.message(500, "failed to establish data connection")  

                    return False  

            return True  

      

        def read_virtual(self, path):  

            vdir_list = []  

            path = path + "/.xxftp/virtual"  

            if os.path.isfile(path):  

                for v in open(path, "r").readlines():  

                    items = v.split()  

                    items[1] = items[1].replace("$root", root_dir)  

                    vdir_list.append(items)  

            return vdir_list  

      

        def get_dir_size(self, folder):  

            size = 0  

            for path, dirs, files in os.walk(folder):  

                for f in files:  

                    size += os.path.getsize(os.path.join(path, f))  

            return size  

      

        def read_size(self, path):  

            size = 0  

            path = path + "/.xxftp/size"  

            if os.path.isfile(path):  

                size = int(open(path, "r").readline())  

            return size  

      

        def read_permission(self, path):  

            permission = "read,write,modify"  

            path = path + "/.xxftp/permission"  

            if os.path.isfile(path):  

                permission = open(path, "r").readline()  

            return permission  

      

        def parse_path(self, path):  

            if path == "": path = "."  

            if path[0] != "/":  

                path = self.curr_dir + "/" + path  

            s = os.path.normpath(path).replace("\\", "/").split("/")  

            local = self.home_dir  

            # reset directory permission  

            vdir_list = self.read_virtual(local)  

            limit_size = self.read_size(local)  

            permission = self.read_permission(local)  

            remote = ""  

            is_virtual = False  

            for name in s:  

                name = name.lstrip(".")  

                if name == "": continue  

                remote = remote + "/" + name  

                is_virtual = False  

                for v in vdir_list:  

                    if v[0] == name:  

                        permission = v[2]  

                        local = v[1]  

                        limit_size = self.read_size(local)  

                        is_virtual = True  

                if not is_virtual: local = local + "/" + name  

                vdir_list = self.read_virtual(local)  

            return (remote, local, permission, vdir_list, limit_size, is_virtual)  

      

        def run(self):  

            ''''' Connection Process '''  

            try:  

                if len(conn_list) > max_connections:  

                    self.message(500, "too many connections!")  

                    self.fd.close()  

                    self.running = False  

                    return  

                # Welcome Message  

                if os.path.exists(root_dir + "/xxftp.welcome"):  

                    self.message(220, open(root_dir + "/xxftp.welcome").read())  

                else:  

                    self.message(220, "xxftp(Python) www.yihaomen.com")  

                # Command Loop  

                line = ""  

                while self.running:  

                    data = self.fd.recv(4096)  

                    if len(data) == 0: break  

                    line += data  

                    if line[-2:] != "\r\n": continue  

                    line = line[:-2]  

                    space = line.find(" ")  

                    if space == -1:  

                        self.process(line, "")  

                    else:  

                        self.process(line[:space], line[space+1:])  

                    line = ""  

            except:  

                print "error", sys.exc_info()  

            self.running = False  

            self.fd.close()  

            print "connection end", self.fd, "user", self.username  

      

        def message(self, code, s):  

            ''''' Send Ftp Message '''  

            s = str(s).replace("\r", "")  

            ss = s.split("\n")  

            if len(ss) > 1:  

                r = (str(code) + "-") + ("\r\n" + str(code) + "-").join(ss[:-1])  

                r += "\r\n" + str(code) + " " + ss[-1] + "\r\n"  

            else:  

                r = str(code) + " " + ss[0] + "\r\n"  

            if self.option_utf8:  

                r = unicode(r, sys.getfilesystemencoding()).encode("utf8")  

            self.fd.send(r)  

      

def server_listen():  

        global conn_list  

        listen_fd = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  

        listen_fd.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  

        listen_fd.bind((listen_ip, listen_port))  

        listen_fd.listen(1024)  

        conn_lock = threading.Lock()  

        print "ftpd is listening on ", listen_ip + ":" + str(listen_port)  

      

        while True:  

            conn_fd, remote_addr = listen_fd.accept()  

            print "connection from ", remote_addr, "conn_list", len(conn_list)  

            conn = FtpConnection(conn_fd)  

            conn.start()  

      

            conn_lock.acquire()  

            conn_list.append(conn)  

            # check timeout  

            try:  

                curr_time = time.time()  

                for conn in conn_list:  

                    if int(curr_time - conn.alive_time) > conn_timeout:  

                        if conn.running == True:  

                            conn.fd.shutdown(socket.SHUT_RDWR)  

                        conn.running = False  

                conn_list = [conn for conn in conn_list if conn.running]  

            except:  

                print sys.exc_info()  

            conn_lock.release()  

      

def main():  

    server_listen()  

      

if __name__ == "__main__":  

    main() 

   

```
