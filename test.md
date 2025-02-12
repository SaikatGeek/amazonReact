all: system_prerequisites create config_ips setup_routing iptables_rules ping

system_prerequisites:
	# sudo apt update && sudo apt upgrade -y
	# sudo apt install iproute2 net-tools tcpdump -y

create:
	# Create network bridges
	sudo ip link add br0 type bridge
	sudo ip link add br1 type bridge
	sudo ip link set br0 up
	sudo ip link set br1 up

	# Verify bridges
	ip link show br0
	ip link show br1

	# Create network namespaces
	sudo ip netns add ns1
	sudo ip netns add ns2
	sudo ip netns add router-ns

	# Verify namespaces
	sudo ip netns list

	# Create virtual interfaces and connect them
	sudo ip link add veth-ns1 type veth peer name veth-br0
	sudo ip link add veth-r-ns1 type veth peer name veth-br0-ns1
	sudo ip link add veth-ns2 type veth peer name veth-br1
	sudo ip link add veth-r-ns2 type veth peer name veth-br1-ns2

	sudo ip link set veth-ns1 netns ns1
	sudo ip link set veth-ns2 netns ns2
	sudo ip link set veth-r-ns1 netns router-ns
	sudo ip link set veth-r-ns2 netns router-ns

	sudo ip link set veth-br0 master br0
	sudo ip link set veth-br0-ns1 master br0
	sudo ip link set veth-br1 master br1
	sudo ip link set veth-br1-ns2 master br1

	sudo ip link set veth-br0 up
	sudo ip link set veth-br0-ns1 up
	sudo ip link set veth-br1 up
	sudo ip link set veth-br1-ns2 up

	# Verify interfaces
	sudo ip netns exec ns1 ip link show
	sudo ip netns exec ns2 ip link show
	sudo ip netns exec router-ns ip link show
	bridge link show

config_ips:
	# Configure IP addresses
	sudo ip netns exec ns1 ip link set veth-ns1 up
	sudo ip netns exec ns1 ip addr add 10.11.0.10/24 dev veth-ns1

	sudo ip netns exec ns2 ip link set veth-ns2 up
	sudo ip netns exec ns2 ip addr add 10.11.0.6/24 dev veth-ns2

	sudo ip netns exec router-ns ip link set veth-r-ns1 up
	sudo ip netns exec router-ns ip addr add 10.11.0.2/24 dev veth-r-ns1
	sudo ip netns exec router-ns ip link set veth-r-ns2 up
	sudo ip netns exec router-ns ip addr add 10.11.0.3/24 dev veth-r-ns2

	sudo ip addr add 10.11.0.11/24 dev br0
	sudo ip addr add 10.11.0.4/24 dev br1

	# Verify IPs
	sudo ip netns exec ns1 ip addr show
	sudo ip netns exec ns2 ip addr show
	sudo ip netns exec router-ns ip addr show
	ip addr show br0
	ip addr show br1

setup_routing:
	# Configure routing
	sudo ip netns exec ns1 ip route add default via 10.11.0.11
	sudo ip netns exec ns2 ip route add default via 10.11.0.4

	# Router default IP
	sudo ip netns exec router-ns ip route add default via 10.11.0.11

	# Enable IP forwarding
	sudo sysctl -w net.ipv4.ip_forward=1
	sudo ip netns exec router-ns sysctl -w net.ipv4.ip_forward=1

	# Verify routing
	sudo ip netns exec ns1 ip route show
	sudo ip netns exec ns2 ip route show
	sudo ip netns exec router-ns ip route show

iptables_rules:
	sudo iptables --append FORWARD --in-interface br0 --jump ACCEPT
	sudo iptables --append FORWARD --out-interface br0 --jump ACCEPT
	sudo iptables --append FORWARD --in-interface br1 --jump ACCEPT
	sudo iptables --append FORWARD --out-interface br1 --jump ACCEPT

ping:
	# Test connectivity
	sudo ip netns exec ns1 ping -c 2 10.11.0.2
	sudo ip netns exec router-ns ping -c 2 10.11.0.10
	sudo ip netns exec ns1 ping -c 2 10.11.0.3
	sudo ip netns exec ns1 ping -c 2 10.11.0.6
	sudo ip netns exec ns2 ping -c 2 10.11.0.3
	sudo ip netns exec router-ns ping -c 2 10.11.0.6
	sudo ip netns exec ns2 ping -c 2 10.11.0.2
	sudo ip netns exec router-ns ping -c 2 10.11.0.10

clean:
	# Clean up
	sudo ip netns del ns1
	sudo ip netns del ns2
	sudo ip netns del router-ns
	sudo ip link del br0
	sudo ip link del br1
