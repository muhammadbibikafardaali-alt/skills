# Skills and notes

Notes on the tools, commands, and topics I work with day to day. Mostly Linux,
some AWS and security. I add to this as I learn things.

## Linux

### updating packages (Debian/Ubuntu)

    sudo apt update        # refresh the list of available packages
    sudo apt upgrade       # install the newer versions
    sudo apt full-upgrade  # upgrade and allow adding/removing deps
    sudo apt autoremove    # drop packages that are no longer needed

update just refreshes the list, upgrade actually installs. easy to mix up.

### file permissions

    ls -l                  # see permissions, owner, group
    chmod 755 script.sh    # rwx for owner, r-x for group and others
    chmod u+x script.sh    # add execute for the owner only
    chmod -R 644 docs/     # recursive
    chown user:group file  # change owner / group

read = 4, write = 2, execute = 1. three digits = owner / group / others.
so 755 means owner rwx (7), group r-x (5), others r-x (5).

### find

    find /var/log -name "*.log"
    find . -type f -size +100M
    find . -type d -name "node_modules"
    find . -mtime -7                          # changed in the last 7 days
    find . -name "*.tmp" -delete
    find . -type f -exec grep -l "TODO" {} \;

### disk usage

    du -sh /home/user           # total size of a folder
    du -h --max-depth=1 /var    # size per subfolder, one level down
    du -ah . | sort -rh | head  # biggest things in here
    df -h                       # free space per filesystem

du = space used by files/folders. df = space left on the disk.

### other commands I keep reaching for

    grep -rn "keyword" .
    ps aux | grep nginx
    top  /  htop
    systemctl status sshd
    journalctl -u sshd -n 50
    tar -czf backup.tar.gz dir/
    scp file user@host:/path

## AWS / cloud

DDoS mitigation (graduation project) - detection + automated response on AWS:

- Shield for network/transport layer DDoS protection
- CloudWatch for metrics, alarms, dashboards to catch traffic spikes
- attack types: volumetric, protocol, application-layer (L7), and matching
  each to the right mitigation (Shield, rate limiting, WAF, alarms)

services used / learning: IAM, CloudWatch, Shield, WAF, S3, VPC, security groups.
next up: GuardDuty, CloudTrail, KMS, Config.

## Security

- DDoS attack types and mitigation (see AWS notes above)
- payment security from the day job - checkout flows, integration auth,
  webhooks; learning to look at them for replay / abuse / input-trust issues
- basics: SQLi, auth flows, working toward web and API security

## Python / scripting

- python for automation scripts and small tools (write with reference)
- bash for system tasks
- git for version control

## todo / learning

- AWS security services (GuardDuty, CloudTrail, KMS)
- PortSwigger web security labs
- threat modeling (STRIDE)
- docker / container security
- AWS Certified Security - Specialty

## changelog

- 2026-05-27  first version: linux, aws/ddos, security, scripting notes
