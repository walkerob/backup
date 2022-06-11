# to BUILD:
#   docker build -t push .
#
# to RUN:
#   docker run -d --restart always --name push -v /etc/localtime:/etc/localtime:ro -v /location/on/NAS/for/collected/files/:/backup -v /location/on/NAS/for/logs/:/logs push
# to STOP, without auto restart:
#   docker kill push
# while running, to view logs:
#   docker logs -f push
#   (or, on NAS, tail -f /location/on/NAS/for/logs/push.log)
# while running, to hack around:
#   docker exec push ls -l
#
# to override ENTRYPOINT:
#   docker run --rm -it --entrypoint /bin/bash -v /etc/localtime:/etc/localtime:ro -v /location/on/NAS/for/collected/files/:/backup -v /location/on/NAS/for/logs/:/logs push
#
FROM ubuntu:latest

# inspiration from https://stackoverflow.com/questions/37458287/how-to-run-a-cron-job-inside-a-docker-container
RUN DEBIAN_FRONTEND=noninteractive apt-get -q update && \
        apt-get install --no-install-recommends -y \
            gawk \
            cron \
# the following required for rclone download
            curl \
            ca-certificates && \
        apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /root

# Download and install rclone
RUN tmp=$(mktemp) && curl https://downloads.rclone.org/v1.58.1/rclone-v1.58.1-linux-amd64.deb >${tmp} && dpkg -i ${tmp} ; rm -v ${tmp}

# config and script
COPY buckets try_push ./

# must be 0644
COPY crontab /etc/cron.d/

# Apply cron job
RUN crontab /etc/cron.d/crontab

COPY rclone.conf ./.config/rclone/

# Run the command on container startup
ENTRYPOINT cron && echo cron running at `date` && touch /logs/push.log && tail -0f /logs/push.log
