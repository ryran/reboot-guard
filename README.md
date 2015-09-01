# reboot-guard
Block systemd-initiated poweroff/reboot/halt until configurable condition checks pass


### Requirements

- Python 2.7
- systemd
- Can be launched from a simple systemd service or run manually


### Interactive walk-thru: ONE-SHOT

1. Download and execute `rguard` without any condition checks to confirm it really can block shutdown.

    ```
    [root]# cd /usr/sbin
    [root]# curl -kO https://raw.githubusercontent.com/ryran/reboot-guard/master/rguard
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 16575  100 16575    0     0  54536      0 --:--:-- --:--:-- --:--:-- 54702
    [root]# ll rguard
    -rwxr-xr-x. 1 root root 16575 Sep  1 02:46 rguard
    [root]# rguard -1
    WARNING: ☹  Blocked poweroff.target
    WARNING: ☹  Blocked reboot.target
    WARNING: ☹  Blocked halt.target
    WARNING: Exiting due to -1 or -0 option
    [root]# 
    ```

1. Try to reboot/shutdown/halt and fail. Note that the old legacy commands (`halt`, `shutdown`, `reboot`) trigger wall messages regardless.

    ```
    [root]# systemctl reboot
    Failed to issue method call: Operation refused, unit reboot.target may be requested by dependency only.
    [root]# shutdown -h now
    Failed to issue method call: Operation refused, unit poweroff.target may be requested by dependency only.

    Broadcast message from root@localhost on pts/0 (Mon 2015-08-31 02:28:05 EDT):

    The system is going down for power-off NOW!
    [root]#
    ```

1. Disable the blocks.

    ```
    [root]# rguard -0
    WARNING: ☻  Unblocked poweroff.target
    WARNING: ☻  Unblocked reboot.target
    WARNING: ☻  Unblocked halt.target
    WARNING: Exiting due to -1 or -0 option
    [root]# 
    ```

### Interactive walk-thru: CONDITION CHECKS

1. Execute `rguard` with a tiny sleep-interval and 2 simple checks.

    ```
    [root]# rguard --interval 15 --unit atd --require-file /run/.require &
    [1] 23317
    WARNING: ☹  Failed a condition-check
    WARNING: ☹  Blocked poweroff.target
    WARNING: ☹  Blocked reboot.target
    WARNING: ☹  Blocked halt.target
    [root]# 
    ```

1. The conditions we set meant shutdown will be blocked while crond is active and while `/root/require` does not exist, so fix that and `rguard` will immediately unblock shutdown and exit.

    ```
    [root]# systemctl is-active atd
    active
    [root]# systemctl stop atd
    [root]# touch /run/.require
    WARNING: ☻  Passed all condition-checks
    WARNING: ☻  Unblocked poweroff.target
    WARNING: ☻  Unblocked reboot.target
    WARNING: ☻  Unblocked halt.target
    [root]# fg
    rguard --interval 15 --unit atd --require-file /run/.require
    ^C
    WARNING: Gracefully exiting due to receipt of SIG 17
    ```


### Instructions

1. Save `rguard` to `/usr/sbin/`
1. See help page and examples in `rguard.service`
1. Play with the options until you get them how you want them
1. Modify `rguard.service` with your options and save it to `/etc/systemd/system/`
1. Run: `systemctl daemon-reload; systemctl enable rguard; systemctl start rguard`
1. Check `systemctl status rguard -n50` or use `journalctl -fu rguard` to keep an eye on logs (turn up the `--loglevel` if necesary)
1. Tweak `rguard.service` as required, and then re-run `systemctl daemon-reload; systemctl restart rguard`
1. Give feedback by posting to the [Issue Tracker](https://github.com/ryran/reboot-guard/issues)


### Help page

```
usage: rguard [-1 | -0] [-f FILE] [-F FILE] [-u UNIT] [-c CMD] [-a ARGS]
              [-r COMMAND] [-h] [-i SEC] [-n] [-x]
              [-v {debug,info,warning,error}] [-t]

Block systemd-initiated shutdown until configurable condition checks pass

SET AND QUIT:
  Execute a single action with no condition checks.

  -1, --install-guard   Install reboot-guard and immediately exit
  -0, --remove-guard    Remove reboot-guard (if present) and immediately exit

CONFIGURE CONDITIONS:
  Establish what condition checks must pass to allow shutdown. Each option may
  be specified multiple times.

  -f, --forbid-file FILE
                        Prevent shutdown while FILE exists
  -F, --require-file FILE
                        Prevent shutdown until FILE exists
  -u, --unit UNIT       Prevent shutdown while systemd UNIT is active
  -c, --cmd CMD         Prevent shutdown while CMD exactly matches at least 1
                        running process
  -a, --args ARGS       Prevent shutdown while ARGS exactly matches the full
                        cmd+args of at least 1 running process
  -r, --run COMMAND     Prevent shutdown while execution of COMMAND fails;
                        prefix COMMAND with '!' to prevent shutdown while
                        execution succeeds (MORE: Prefix COMMAND with '@' to
                        execute it as a shell command, e.g., to use pipes '|'
                        or other logic; examples: '@cmd|cmd' or '!@cmd|cmd')

OPTIONS:

  -h, --help            Show this help message and exit
  -i, --interval SEC    Modify the sleep interval between condition checks
                        (default: 60 seconds)
  -n, --ignore-signals  Ignore the most common signals (HUP, INT, QUIT, USR1,
                        USR2, TERM), in which case a graceful exit requires
                        the next option or else SIGKILL
  -x, --exit-on-pass    Exit (and remove reboot-guard) the first time all
                        condition checks pass
  -v, --loglevel {debug,info,warning,error}
                        Specify minimum message type to print (default:
                        warning)
  -t, --timestamps      Enable timestamps in message output (not necessary if
                        running as systemd service)
```
