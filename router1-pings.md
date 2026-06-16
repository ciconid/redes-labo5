    for target in \
    "192.168.13.1    router2-Red4" \
    "192.168.13.129  router4-Red5" \
    "192.168.13.193  router2-Red6" \
    "192.168.13.194  router3-Red6" \
    "192.168.13.201  router3-Red9" \
    "172.31.0.1      router3-Red31" \
    "192.168.13.197  router2-Red7" \
    "192.168.13.198  router4-Red7" \
    "192.168.13.202  router4-Red9" \
    "192.168.12.1    router4-Red8" \
    "172.31.0.254    router_internet-Red31" \
    "8.8.8.8         Internet"; do
    ip=$(echo $target | awk '{print $1}')
    label=$(echo $target | awk '{print $2}')
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "▶ $label ($ip)"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    ping -c2 $ip
    echo ""
    done