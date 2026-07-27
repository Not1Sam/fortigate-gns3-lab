FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "echo 'interface=eth0' > /etc/dnsmasq.conf && echo 'dhcp-range=192.168.10.100,192.168.10.200,12h' >> /etc/dnsmasq.conf && echo 'dhcp-option=3,192.168.10.1' >> /etc/dnsmasq.conf && echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && dnsmasq --no-daemon"]
