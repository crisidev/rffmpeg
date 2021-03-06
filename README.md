# rffmpeg

rffmpeg is a remote FFmpeg wrapper used to execute FFmpeg commands on a remote server via SSH. It is most useful in situations involving media servers such as Jellyfin (our reference user), where one might want to perform transcoding actions with FFmpeg on a remote machine or set of machines to better handle the load.

## Usage

1. Install the required Python 3 dependencies: `yaml` and `subprocess`.

1. Create the directory `/etc/rffmpeg`.

1. Copy the `rffmpeg.yml.sample` file to `/etc/rffmpeg/rffmpeg.yml` and edit it to suit your needs.

1. Install `rffmpeg.py` somewhere useful, for instance at `/usr/local/bin/rffmpeg.py`.

1. Create symlinks for the command names `ffmpeg` and `ffprobe` to `rffmpeg.py`, for instance `/usr/local/bin/ffmpeg -> /usr/local/bin/rffmpeg.py` and `/usr/local/bin/ffprobe -> /usr/local/bin/rffmpeg.py`.

1. Edit your media program to use the `rffmpeg.py` binary (via the symlink names) instead of the standard `ffmpeg` binary.

1. Profit!

## rffmpeg options and caveats

### Remote hosts

rffmpeg supports setting multiple hosts. It keeps state in `/run/shm/rffmpeg`, of all running processes. These state files are used during rffmpeg's initialization in order to determine the optimal target host. rffmpeg will run through these hosts sequentially, choosing the one with the fewest running rffmpeg jobs. This helps distribute the transcoding load across multiple servers, and can also provide redundancy if one of the servers is offline - rffmpeg will detect if a host is unreachable and set it "bad" for the remainder of the run, thus skipping it until the process completes.

### Local host
If one of the hosts in the config file is called "localhost", rffmpeg will run locally without SSH. This can be usefull if you have a host which serves the frontend (IE Jellyfin) and is also able to transcode.

### Terminating rffmpeg

When running rffmpeg manually, *do not* exit it with `Ctrl+C`. Doing so will likely leave the `ffmpeg` process running on the remote machine. Instead, enter `q` and a newline ("Enter") into the rffmpeg process, and this will terminate the entire command cleanly. This is the method that Jellyfin uses to communicate the termination of an `ffmpeg` process.

### Local fallback

rffmpeg will fall back to a local copy of ffmpeg, at the same location as on remote systems (i.e. as configured in `/etc/rffmpeg/rffmpeg.yml`), should it be unable to find any working remote hosts. This helps prevent situations where rffmpeg cannot be run due to none of the remote host(s) being unavailable. Note that, if hardware acceleraton is configured and is not available locally, this may still fail; there is no easy way around this.

If you want the local system to be included in the normal list, for instance if the local system is also a powerful transcode machine, you can add `localhost` to the list of hosts in order to have it be used along with the remote systems; it will SSH to itself but, if the guide below is followed exactly, will work as expected.

## Full setup guide

This example setup is the one I use for `rffmpeg`, involving a media server (`jf1`) and a remote transcode server (`gpu1`). Both systems run Debian GNU/Linux, though the commands below should also work on Ubuntu. Note that Docker is not officially supported with `rffmpeg` due to the complexity of exporting Docker volumes with NFS, the path differences, and the fact that I don't use Docker, but if you do figure it out a PR is welcome.

1. Prepare the media server (`jf1`) with Jellyfin using the standard `.deb` install. Make note of main Jellyfin data path; it's usually `/var/lib/jellyfin` unless you change it. Note that if you change this path, or put the various subdirectories such as the `transcodes` or `data/subtitles` directories elsewhere, you may need to alter the NFS steps below to accommodate this.

1. On the media server, create an SSH keypair owned by the Jellyfin service user; save this SSH key somewhere readable to the service user: `sudo -u jellyfin mkdir -p /var/lib/jellyfin/.ssh && sudo -u jellyfin ssh-keygen -t rsa -f /var/lib/jellyfin/.ssh/id_rsa`.

1. Copy (or symlink) the new SSH public key created in the previous step to `authorized_keys`; this will be used later when the Jellyfin data directory is mounted on the transcode server: `sudo -u jellyfin cp -a /var/lib/jellyfin/.ssh/id_rsa.pub /var/lib/jellyfin/.ssh/authorized_keys`

1. Install the rffmpeg program as detailed in the above section, including creating the `/etc/rffmpeg/rffmpeg.yml` configuration file and symlinks.

1. Install the NFS kernel server: `sudo apt -y install nfs-kernel-server`

1. Export your Jellyfin data path found in step 1 with NFS; you will need to know the local IP address of the transcode server(s) (e.g. `10.0.0.100`) to lock this down; alternatively, use your entire local network range (e.g. `10.0.0.0/24`), though this is not recommended for security reasons: `echo '/var/lib/jellyfin 10.0.0.100/32(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports && sudo systemctl restart nfs-kernel-server`

1. On the transcode server, install any required tools or programs to make use of hardware transcoding; this is optional if you only use software (i.e. CPU) transcoding.

1. Install the `jellyfin-ffmpeg` package to provide an FFmpeg binary; follow the Jellyfin installation instructions for details on setting up the Jellyfin repository, though install only `jellyfin-ffmpeg`.

1. Install the NFS client utilities: `sudo apt install -y nfs-common`

1. Create a user for rffmpeg to SSH into the server as. This user should match the `jellyfin` user on the media server in every way, including UID (`id jellyfin` on the media server), home path, and groups.

1. Ensure that the Jellyfin data directory exists at the same location as on the media server; create it if required, and set it immutable to prevent unintended writes: `sudo mkdir -p /var/lib/jellyfin && sudo chattr +i /var/lib/jellyfin`

1. Mount the media server NFS data share at the same directory on the transcode server: `echo 'jf1:/var/lib/jellyfin /var/lib/jellyfin nfs defaults,vers=3,sync 0 0' | sudo tee -a /etc/fstab && sudo mount -a`

1. Mount your media directory on the transcode server at the same location as on the media server and using the same method; if your media is local to the media server, export it with NFS similarly to the data directory.

1. On the media server, attempt to SSH to the transcode server as the `jellyfin` user using the key from step 2; this both tests the connection as well as saves the transcode server SSH host key locally: `sudo -u jellyfin ssh -i /var/lib/jellyfin/.ssh/id_rsa jellyfin@gpu1`

1. Verify that rffmpeg itself works by calling its `ffmpeg` alias with the `-version` option: `sudo -u jellyfin /usr/local/bin/ffmpeg -version`

1. In Jellyfin, set the rffmpeg binary, via its `ffmpeg` symlink, as your "FFmpeg path" in the Playback settings; optionally, enable any hardware encoding you configured in step 7.

1. Try running a transcode and verifying that the `rffmpeg` program works as expected. The flow should be:

    1. Jellyfin calls rffmpeg with the expected arguments.

    1. FFmpeg begins running on the transcode server.

    1. The FFmpeg process writes the output files to the NFS-mounted temporary transcoding directory.

    1. Jellyfin reads the output files from the NFS-exported temporary transcoding directory and plays back normally.

1. `rffmpeg` will also be used during other tasks in Jellyfin that require `ffmpeg`, for instance image extraction during library scans and subtitle extraction.
