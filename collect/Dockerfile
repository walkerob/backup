# to BUILD:
#   docker build -t collect .
#
# to RUN:
#   docker run -d --restart always --name collect -v /etc/localtime:/etc/localtime:ro -v /location/on/NAS/for/collected/files/:/backup -v /location/on/NAS/for/logs/:/logs --device /dev/fuse --cap-add SYS_ADMIN --security-opt apparmor:unconfined collect
# to STOP, without auto restart:
#   docker kill collect
# while running, to view logs:
#   docker logs -f collect
#   (or, on NAS, tail -f /location/on/NAS/for/logs/collect.log)
# while running, to hack around:
#   docker exec collect ls -l
# e.g. docker exec collect rm -v <touchfile>
#
# to override ENTRYPOINT:
#   docker run --rm -it --entrypoint /bin/bash -v /etc/localtime:/etc/localtime:ro -v /location/on/NAS/for/collected/files/:/backup -v /location/on/NAS/for/logs/:/logs --device /dev/fuse --cap-add SYS_ADMIN --security-opt apparmor:unconfined collect
#
FROM ubuntu:latest

# inspiration from https://stackoverflow.com/questions/37458287/how-to-run-a-cron-job-inside-a-docker-container
RUN DEBIAN_FRONTEND=noninteractive apt-get -q update && \
        apt-get install --no-install-recommends -y \
            gawk \
            cron \
            sshfs \
            iputils-ping \
            rsync && \
        apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /root

# generate new ssh key
RUN cat /dev/zero | ssh-keygen -q -N "" >/dev/null && echo "Copy to authorized_keys:" && cat .ssh/id_rsa.pub

# .ssh exists now - copy in known_hosts
COPY known_hosts .ssh/

# config and script
COPY sources try_collect ./

# fstab for mounting backup source drives
COPY fstab /etc/fstab

# must be 0644
COPY crontab /etc/cron.d/

# Apply cron job
RUN crontab /etc/cron.d/crontab

# Run the command on container startup
ENTRYPOINT cron && echo cron running at `date` && touch /logs/collect.log && tail -0f /logs/collect.log
