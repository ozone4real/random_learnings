# NFS

NFS (Network file system) is a protocol used to mount and synchronize remote file systems to a local file system over the network. It is used by cloud service providers to provide durable and expandable storage volumes for databases, web servers, etc.
The protocol involves a client and a server.

## NFS Server

There are 3 critical components to an NFS server:

1. rpcbind: This service is responsible for mapping client requests to the appropriate NFS service. When a client wants to connect to
an RPC service like NFS, it first contacts rpcbind to find out which port the service is running on. The rpcbind service listens on port 111.
2. mountd: This service is responsible for managing mount requests from clients. It handles the mounting of remote file systems and ensures that the client has the necessary permissions to access the shared directories. It does this by checking the export list defined in the `/etc/exports` file. mountd listens on a port that is dynamically assigned by rpcbind but you can specify a static port in the `/etc/default/nfs-kernel-server` file or in the `/etc/nfs.conf` file (more on these later). Assigning a static port is useful if the remote host is behind a firewall, so the static port can be white-listed in the firewall config.
3. nfsd: This is the actual NFS server daemon that handles the file system operations (read, write, etc.) for the mounted file systems and serving files over the network. It typically runs on port 2049 but you could also specify a static port in the `/etc/nfs.conf` file. nfsd is launched by the `nfs-kernel-server` service and can be configured to run multiple threads to handle concurrent requests.

### Exports configuration (see: <https://www.man7.org/linux/man-pages/man5/exports.5.html>)

The `/etc/exports` is used to configure directories on the server that should be accessible by specific clients. The file contains an entry for each directory:clients mapping and access control and security configurations for the entry. An entry has the following format:

`directory` `specific client ip or ip range or wildcard (*)`(list of options)

### Examples

1. Expose /srv/mounts to client 120.10.20.2, allow the client read and write (rw) and reply to client requests only after the changes have been flushed to disk (sync). More on the other options later.

 ```sh
 /srv/mounts 120.10.20.2(rw,sync,no_subtree_check,no_root_squash)
 ```

2. Same as above but a directory is mapped to multiple clients

  ```sh
  /srv/mounts 120.10.20.2(rw,sync,no_subtree_check,no_root_squash) 35.210.20.2(rw,sync,no_subtree_check,no_root_squash)
  ```

3. Expose /srv/mounts to all clients in the 120.10.20.0/24 network

  ```sh
  /srv/mounts 120.10.20.0/24(rw,sync,no_subtree_check,no_root_squash)
  ```

4. Expose /srv/mounts to all clients.

  ```sh
  /srv/mounts *(rw,sync,no_subtree_check,no_root_squash)
  ```

Common options and their meaning:

- rw: allow read and write operations on the directory.
- sync (default): Reply to requests only after file changes have been flushed to disk.
- async: Reply to requests immediately changes are saved at the OS file system level (before the changes have been flushed to disk).
- secure (default): Only allow client connections from priviledged ports (< 1024).
- insecure: Allow connections from any port.
- root_squash (default): This maps client requests from user 0 (i.e root user) to anonymous/nobody uid. This is a security measure that prevents root users in the client from performing file operations on the server as root.
- no_root_squash: Turns off root_squash
- subtree_check: This always verifies that the file/directory being accessed over NFS is in the tree of the exported directory. This could can cause problems with accessing files that are renamed while being opened on a client.
- no_subtree_check: Turns off subtree_check
- all_squash: Maps client requests from all users to anonymous/nobody uid. Useful for NFS-exported public FTP directories, news spool directories, etc.
- no_all_squash (default): Turns off all_squash
- anonuid and anongid: Explicitly set a uid and gid for the anonymous/nobody user. This option is primarily useful for
NFS clients, where you might want all requests appear to be from one user.

### Lifecycle of an NFS request

1. A client sends a request to the rpcbind service on the server to find out which port the NFS services are running on.
2. The rpcbind service responds with the port numbers for the mountd and nfsd services.
3. The client then sends a mount request to the mountd service to mount the desired remote file system.
4. The mountd service checks the `/etc/exports` file to verify if the client has permission to access the requested file system.
5. If the client is allowed, a file handle (unique identifier) is returned to the client for use in subsequent requests.
6. Through an NFS client module, the client can now access the mounted file system as if it were a local file system.
7. The NFS client module translates local file operations (read, write, etc.) to NFS requests. It sends requests with the file handle directly to the nfsd service on the server, which processes the requests and returns the results to the client.

### Setting up the NFS Server

1. Install the `nfs-kernel-server` library:

  ```sh
    sudo apt install nfs-kernel-server
  ```

2. Setup an NFS export list of directories you want accessible to specific clients in the /etc/exports file (see exports(5))
    /srv/mounts 120.10.20.2(rw,sync,no_subtree_check,root_squash)
3. Configure a fixed port for mountd in the `/etc/nfs.conf` file (under [mountd]), e.g 20048
4. Start the nfs-kernel-server

    ```
    sudo systemctl restart nfs-kernel-server
    ```

5. Expose the mountd port (what you configured in step 3.), the nfsd port (typically 2049) and the rpc.bind port (111) in your server's firewall
6. Verify that the services are running on the configured/expected ports

    ```
    sudo netstat -tulpn | egrep 'rpc.mountd|2049|111'
    ```

### NFS Client
