# Golink compose file

[Github Repo](https://github.com/tailscale/golink)

*Make sure to change get an auth key from your tailscale admin console
and place it in [./.env].*

*Change the volume location if needed
depending on where you would like to store the container's data and configs.*

You need to add `:Z` to the volumes so it works correctly
without asking for permissions on SELinux like fedora server wiht podman.
