# Script to deploy an Azure Hub Vnet with no Spokes and a Azure VPN Connection with no Local Network Gateways and BGP enabled

```bash
#!/bin/bash

# Collect input parameters from the user

echo -n "Please enter a resource group name: "
read RGNAME
echo -n "Please enter an Azure region (i.e. \"eastus\", \"westus\"):"
read LOCATION
echo -n "Please enter a password for the VMs:"
read ADMIN_PASSWORD

# Variables (change location as relevant)
rg=$RGNAME 
location=$LOCATION
username=azureuser
adminpassword=$ADMIN_PASSWORD
vnet_name=hub
vnet_prefix=10.2.0.0/16
vnet_prefix_long='10.2.0.0 255.255.0.0'
hub_vm_subnet_name=vm
hub_vm_subnet_prefix=10.2.10.0/24
gw_subnet_prefix=10.2.0.0/24


# Azure VPN GW
vpngw_name=vpngw
vpngw_asn=64000
vpngw_pip1="${vpngw_name}-pip1"
vpngw_pip2="${vpngw_name}-pip2"

# Create Vnet
echo "Creating RG and VNet..."
az group create -n $rg -l $location -o none
az network vnet create -g $rg -n $vnet_name --address-prefix $vnet_prefix --subnet-name $hub_vm_subnet_name --subnet-prefix $hub_vm_subnet_prefix -o none
az network vnet subnet create -n GatewaySubnet --address-prefix $gw_subnet_prefix --vnet-name $vnet_name -g $rg -o none

# Create test VM in hub
az vm create -n hubvm -g $rg -l $location --image ubuntuLTS  \
    --admin-username "$username" \
    --admin-password "$adminpassword" \
    --public-ip-address hubvm-pip --vnet-name $vnet_name --size Standard_B1s --subnet $hub_vm_subnet_name -o none

# hub_vm_ip=$(az network public-ip show -n hubvm-pip --query ipAddress -o tsv -g $rg) && echo $hub_vm_ip
# hub_vm_nic_id=$(az vm show -n hubvm -g "$rg" --query 'networkProfile.networkInterfaces[0].id' -o tsv) && echo $hub_vm_nic_id
# hub_vm_private_ip=$(az network nic show --ids $hub_vm_nic_id --query 'ipConfigurations[0].privateIpAddress' -o tsv) && echo $hub_vm_private_ip


# Create VPN Gateway (IP Sec Tunnel to be established with on-prem)

echo "Creating vnet gateway. command will finish running but gw creation takes a while"

az network public-ip create -n $vpngw_pip1 -g $rg --allocation-method Dynamic
az network public-ip create -n $vpngw_pip2 -g $rg --allocation-method Dynamic
az network vnet-gateway create -n $vpngw_name -l $location --public-ip-addresses $vpngw_pip1 $vpngw_pip2 -g $rg --vnet $vnet_name --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --no-wait

#=======================================================
# Check every second for Succeeded
#=======================================================
declare lookfor='"Succeeded"'
declare cmd="az network vnet-gateway list -g $rg | jq '.[0].provisioningState'"
gwStatus=$(eval $cmd)
echo "Gatway provisioning state = $gwStatus"
while [ "$gwStatus" != "$lookfor" ]
do
    sleep 10
    echo "Gatway provisioning state not equal to Succeeded, instead = $gwStatus"
    gwStatus=$(eval $cmd)
    echo "Gateway status = "$gwStatus
done
echo "Gateway Created"

#=======================================================
# Check every second for Succeeded when enabling BGP
#=======================================================
# Enable BGP
az network vnet-gateway update -g $rg -n $vpngw_name --enable-bgp true --no-wait

gwStatus=$(eval $cmd)
echo "Gatway provisioning state = $gwStatus"
while [ "$gwStatus" != "$lookfor" ]
do
    sleep 10
    echo "Gatway provisioning state not equal to Succeeded, instead = $gwStatus"
    gwStatus=$(eval $cmd)
    echo "Gateway status = "$gwStatus
done
```
