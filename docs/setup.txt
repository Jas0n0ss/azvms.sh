#!/bin/bash
set -e

# ============================================================
# Colors
# ============================================================
GREEN="\e[32m"
YELLOW="\e[33m"
RED="\e[31m"
BLUE="\e[34m"
PURPLE="\e[35m"
CYAN="\e[36m"
RESET="\e[0m"

log_info()   { printf "${BLUE}>>> %s${RESET}\n" "$1"; }
log_warn()   { printf "${YELLOW}>>> %s${RESET}\n" "$1"; }
log_error()  { printf "${RED}>>> %s${RESET}\n" "$1"; }
log_vm()     { printf "${GREEN}%s${RESET}\n" "$1"; }
log_res()    { printf "${PURPLE}%s${RESET}\n" "$1"; }
log_confirm(){ printf "${CYAN}%s${RESET} " "$1"; }

# ============================================================
# Help
# ============================================================
show_help() {
    printf "${GREEN}Usage:${RESET} %s [create|remove|--help]\n\n" "$0"
    printf "  ${BLUE}create${RESET}  : Create VM(s) interactively\n"
    printf "  ${BLUE}remove${RESET}  : Delete VM(s) interactively\n"
    printf "  ${BLUE}--help${RESET}  : Show this help message\n"
    exit 0
}

# ============================================================
# default configuration
# ============================================================
DEFAULT_RG="nluatpoc"
DEFAULT_LOCATION="uaenorth"
DEFAULT_VNET_NAME="poc-vnet"
DEFAULT_SUBNET_NAME="poc-subnet"

VM_SIZES_NAMES=("1:Standard_D4s_v3:4c4g" "2:Standard_D4s_v3:4c8g" "3:Standard_D8s_v3:8c16g")
IMAGES=("RedHat|RHEL|9-lvm-gen2|latest" "OpenLogic|CentOS|8_2-gen2|latest" "canonical|Ubuntu|22_04-lts-gen2|latest")
DEFAULT_SSH_KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFsj4hS9dVLi+u3FmUpMkyS2Dz+YpDTwq1S197lC2GQ7 hubo@Mac"

# ============================================================
# Argument check
# ============================================================
if [ $# -eq 0 ]; then show_help; fi
ACTION="$1"; shift || true

case "$ACTION" in
    --help) show_help ;;
    create|remove) ;;
    *) log_error "Unknown option: $ACTION"; show_help ;;
esac

# ============================================================
# Helper functions
# ============================================================
confirm() {
    read -rp "$(log_confirm "$1 [y/N]:")" ans
    case "$ans" in
        [Yy]*) return 0 ;;
        *) return 1 ;;
    esac
}

list_rg_vms() {
    az vm list -g "$DEFAULT_RG" --query "[].name" -o tsv
}

generate_init_script() {
    local OS="$1"
    case "$OS" in
        CentOS)
cat <<'EOF'
#!/bin/bash
echo ">>> [init] Starting CentOS initialization"
PS1="[\u@\h \W]\$ "
mkdir -p /etc/yum.repos.d/bak
KEEP_KEYWORDS="OpenLogic"
for file in /etc/yum.repos.d/*; do
  fname=$(basename "$file")
  [[ "$fname" =~ $KEEP_KEYWORDS ]] && echo ">>> [repo] Keeping $fname" || mv "$file" /etc/yum.repos.d/bak/
done
cat >/etc/yum.repos.d/epel.repo <<'EOR'
[epel]
name=EPEL Everything 9 - $basearch
baseurl=https://dl.fedoraproject.org/pub/epel/9/Everything/$basearch/
enabled=1
gpgcheck=0
EOR
cat >/etc/yum.repos.d/CentOS-Vault-8.2.repo <<'EOR'
[BaseOS]
name=CentOS-8.2.2004 BaseOS
baseurl=https://vault.centos.org/8.2.2004/BaseOS/$basearch/os/
enabled=1
gpgcheck=0
[AppStream]
name=CentOS-8.2.2004 AppStream
baseurl=https://vault.centos.org/8.2.2004/AppStream/$basearch/os/
enabled=1
gpgcheck=0
[PowerTools]
name=CentOS-8.2.2004 PowerTools
baseurl=https://vault.centos.org/8.2.2004/PowerTools/$basearch/os/
enabled=1
gpgcheck=0
[Extras]
name=CentOS-8.2.2004 Extras
baseurl=https://vault.centos.org/8.2.2004/extras/$basearch/os/
enabled=1
gpgcheck=0
EOR
yum clean all
yum makecache -y
yum install -y wget curl vim net-tools git htop lsof which tar unzip bash-completion
setenforce 0 || true
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld || true
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
echo ">>> [init] CentOS initialization done"
EOF
        ;;
        RHEL)
cat <<'EOF'
#!/bin/bash
echo ">>> [init] Starting RHEL initialization"
PS1="[\u@\h \W]\$ "
yum install -y wget curl vim net-tools git htop lsof which tar unzip bash-completion
setenforce 0 || true
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld || true
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
echo ">>> [init] RHEL initialization done"
EOF
        ;;
        Ubuntu)
cat <<'EOF'
#!/bin/bash
echo ">>> [init] Starting Ubuntu initialization"
PS1="[\u@\h \W]\$ "
apt update -y
apt install -y wget curl vim net-tools git htop lsof unzip bash-completion
echo ">>> [init] Ubuntu initialization done"
EOF
        ;;
        *) log_error "Unsupported OS for init: $OS"; exit 1 ;;
    esac
}

# ============================================================
# CREATE FUNCTION
# ============================================================
create_vms() {
    # 1. 输入资源组、位置、VNet、Subnet
    read -rp "$(printf "${BLUE}Enter Resource Group [${DEFAULT_RG}]: ${RESET}")" RG
    RG="${RG:-$DEFAULT_RG}"

    read -rp "$(printf "${BLUE}Enter Location [${DEFAULT_LOCATION}]: ${RESET}")" LOCATION
    LOCATION="${LOCATION:-$DEFAULT_LOCATION}"

    read -rp "$(printf "${BLUE}Enter VNet name [${DEFAULT_VNET_NAME}]: ${RESET}")" VNET_NAME
    VNET_NAME="${VNET_NAME:-$DEFAULT_VNET_NAME}"

    read -rp "$(printf "${BLUE}Enter Subnet name [${DEFAULT_SUBNET_NAME}]: ${RESET}")" SUBNET_NAME
    SUBNET_NAME="${SUBNET_NAME:-$DEFAULT_SUBNET_NAME}"

    # 2. 输入 VM 名称列表
    read -rp "$(printf "${BLUE}Enter VM names (space-separated): ${RESET}")" VM_NAMES
    [[ -z "$VM_NAMES" ]] && { log_warn "No VM names entered, exiting."; return; }

    # 3. 选择 VM Size
    printf "${BLUE}Select VM size:${RESET}\n"
    for i in "${VM_SIZES_NAMES[@]}"; do
        IFS=":" read -r idx size mem <<< "$i"
        printf "  %s) %s (%s)\n" "$idx" "$size" "$mem"
    done
    read -rp "$(printf "${BLUE}Choice [1-%d]: ${RESET}" "${#VM_SIZES_NAMES[@]}")" size_choice
    VM_SIZE="${VM_SIZES_NAMES[$((size_choice-1))]}"
    VM_SIZE="${VM_SIZE#*:}"
    VM_SIZE="${VM_SIZE%%:*}"

    # 4. 选择 OS Image
    printf "${BLUE}Select OS image:${RESET}\n"
    for i in "${!IMAGES[@]}"; do
        IFS="|" read -r pub offer sku ver <<< "${IMAGES[$i]}"
        printf "  %d) %s %s %s %s\n" "$((i+1))" "$pub" "$offer" "$sku" "$ver"
    done
    read -rp "$(printf "${BLUE}Choice [1-%d]: ${RESET}" "${#IMAGES[@]}")" img_choice
    ((img_choice--))
    IFS="|" read -r IMAGE_PUBLISHER IMAGE_OFFER IMAGE_SKU IMAGE_VERSION <<< "${IMAGES[$img_choice]}"

    # 5. SSH key
    read -rp "$(printf "${BLUE}Enter SSH key (or press Enter to use default): ${RESET}")" SSH_KEY
    SSH_KEY="${SSH_KEY:-$DEFAULT_SSH_KEY}"

    # 6. 总览展示
    printf "${BLUE}============================================================${RESET}\n"
    printf "${BLUE}>>> The following VMs and resources will be created:${RESET}\n"
    printf "${PURPLE}Resource Group: ${RESET}%s\n" "$RG"
    printf "${PURPLE}Location      : ${RESET}%s\n" "$LOCATION"
    printf "${PURPLE}VNet          : ${RESET}%s\n" "$VNET_NAME"
    printf "${PURPLE}Subnet        : ${RESET}%s\n" "$SUBNET_NAME"
    printf "\n"
    for VM_NAME in $VM_NAMES; do
        EXISTING=$(az vm show -g "$RG" -n "$VM_NAME" --query "name" -o tsv 2>/dev/null || true)
        if [ "$EXISTING" = "$VM_NAME" ]; then
            log_warn "VM $VM_NAME already exists, will be skipped."
        else
            log_vm "  Name : $VM_NAME"
            log_res "  Size : $VM_SIZE"
            log_res "  Image: $IMAGE_PUBLISHER/$IMAGE_OFFER/$IMAGE_SKU/$IMAGE_VERSION"
            log_res "  SSH  : ${SSH_KEY:0:40}..."
        fi
    done
    printf "${BLUE}============================================================${RESET}\n"

    # 7. 一次确认
    read -rp "$(printf "${BLUE}Confirm creation of above resources? [y/N]: ${RESET}")" confirm
    [[ ! "$confirm" =~ ^[Yy]$ ]] && { log_info ">>> Creation canceled."; return; }

    # 8. 批量创建
    for VM_NAME in $VM_NAMES; do
        EXISTING=$(az vm show -g "$RG" -n "$VM_NAME" --query "name" -o tsv 2>/dev/null || true)
        [ "$EXISTING" = "$VM_NAME" ] && continue

        INIT_FILE="/tmp/init-$VM_NAME.sh"
        generate_init_script "$IMAGE_OFFER" > "$INIT_FILE"
        chmod +x "$INIT_FILE"

        az vm create \
            --resource-group "$RG" \
            --name "$VM_NAME" \
            --image "${IMAGE_PUBLISHER}:${IMAGE_OFFER}:${IMAGE_SKU}:${IMAGE_VERSION}" \
            --size "$VM_SIZE" \
            --admin-username azureuser \
            --ssh-key-values "$SSH_KEY" \
            --vnet-name "$VNET_NAME" \
            --subnet "$SUBNET_NAME" \
            --location "$LOCATION" \
            --os-disk-size-gb 128 \
            --custom-data "$INIT_FILE" \
            --public-ip-address "" \
            --no-wait \
            --only-show-errors

        log_info ">>> VM creation triggered asynchronously for $VM_NAME."
    done

    printf "${BLUE}============================================================${RESET}\n"
    printf "${BLUE}===========   VM Creation Completed   ======================%s\n" ""
    printf "${PURPLE}>>> Check VM status using: az vm list -d -g %s -o table${RESET}\n" "$RG"
    printf "${BLUE}============================================================${RESET}\n"
}

# ============================================================
# REMOVE FUNCTION
# ============================================================
remove_vms() {
    # 1. 输入资源组、位置、VNet、Subnet
    read -rp "$(printf "${BLUE}Enter Resource Group [${DEFAULT_RG}]: ${RESET}")" RG
    RG="${RG:-$DEFAULT_RG}"

    read -rp "$(printf "${BLUE}Enter Location [${DEFAULT_LOCATION}]: ${RESET}")" LOCATION
    LOCATION="${LOCATION:-$DEFAULT_LOCATION}"

    read -rp "$(printf "${BLUE}Enter VNet name [${DEFAULT_VNET_NAME}]: ${RESET}")" VNET_NAME
    VNET_NAME="${VNET_NAME:-$DEFAULT_VNET_NAME}"

    read -rp "$(printf "${BLUE}Enter Subnet name [${DEFAULT_SUBNET_NAME}]: ${RESET}")" SUBNET_NAME
    SUBNET_NAME="${SUBNET_NAME:-$DEFAULT_SUBNET_NAME}"

    # 2. 获取当前 RG 的 VM 列表
    VMS=($(az vm list -g "$RG" --query "[].name" -o tsv))
    if [ ${#VMS[@]} -eq 0 ]; then
        log_warn "No VMs found in RG $RG, exiting."
        return
    fi

    printf "${BLUE}>>> VMs in RG %s:${RESET}\n" "$RG"
    for i in "${!VMS[@]}"; do
        printf "  %d) %s\n" "$((i+1))" "${VMS[$i]}"
    done

    # 3. 用户输入要删除的 VM 序号
    while true; do
        read -rp "$(printf "${CYAN}Enter VM numbers to delete (space-separated, e.g., 1 3 5): ${RESET}")" vm_choices
        VALID_CHOICES=()
        for num in $vm_choices; do
            if [[ "$num" =~ ^[0-9]+$ ]] && [ "$num" -ge 1 ] && [ "$num" -le "${#VMS[@]}" ]; then
                VALID_CHOICES+=("${VMS[$((num-1))]}")
            fi
        done
        if [ ${#VALID_CHOICES[@]} -eq 0 ]; then
            log_warn "No valid VMs selected, please enter valid numbers."
        else
            break
        fi
    done

    # 4. 展示总览
    printf "${BLUE}============================================================${RESET}\n"
    printf "${BLUE}>>> The following VMs and associated resources will be deleted:${RESET}\n"
    printf "${PURPLE}Resource Group: ${RESET}%s\n" "$RG"
    printf "${PURPLE}Location      : ${RESET}%s\n" "$LOCATION"
    printf "${PURPLE}VNet          : ${RESET}%s\n" "$VNET_NAME"
    printf "${PURPLE}Subnet        : ${RESET}%s\n" "$SUBNET_NAME"
    printf "\n"
    for VM_NAME in "${VALID_CHOICES[@]}"; do
        NIC=$(az vm show -g "$RG" -n "$VM_NAME" --query "networkProfile.networkInterfaces[0].id" -o tsv)
        PUBIP=$(az network nic show --ids "$NIC" --query "ipConfigurations[0].publicIpAddress.id" -o tsv 2>/dev/null || true)
        DISK=$(az vm show -g "$RG" -n "$VM_NAME" --query "storageProfile.osDisk.managedDisk.id" -o tsv)
        NSG=$(az network nic show --ids "$NIC" --query "networkSecurityGroup.id" -o tsv 2>/dev/null || true)

        log_vm "  VM: $VM_NAME"
        log_res "    NIC      : $NIC"
        log_res "    Public IP: $PUBIP"
        log_res "    Disk     : $DISK"
        log_res "    NSG      : $NSG"
    done
    printf "${BLUE}============================================================${RESET}\n"

    # 5. 一次确认
    read -rp "$(printf "${CYAN}Confirm deletion of above resources? [y/N]: ${RESET}")" confirm
    [[ ! "$confirm" =~ ^[Yy]$ ]] && { log_info ">>> Deletion canceled."; return; }

    # 6. 批量删除
    for VM_NAME in "${VALID_CHOICES[@]}"; do
        log_info ">>> Deleting VM $VM_NAME..."
        az vm delete -g "$RG" -n "$VM_NAME" --yes --no-wait --only-show-errors

        if [ -n "$NIC" ]; then
            az network nic delete --ids "$NIC" --only-show-errors 2>/dev/null
        fi
        if [ -n "$PUBIP" ]; then
            az network public-ip delete --ids "$PUBIP" --only-show-errors 2>/dev/null
        fi
        if [ -n "$DISK" ]; then
            az disk delete --ids "$DISK" --yes --only-show-errors 2>/dev/null
        fi
        if [ -n "$NSG" ]; then
            az network nsg delete --ids "$NSG" --only-show-errors 2>/dev/null
        fi
        log_info ">>> VM $VM_NAME and associated resources deletion triggered."
    done

    printf "${BLUE}============================================================${RESET}\n"
    printf "${BLUE}===========   VM Deletion Completed   =====================%s\n" ""
    printf "${PURPLE}>>> Check VM status using: az vm list -d -g %s -o table${RESET}\n" "$RG"
    printf "${BLUE}============================================================${RESET}\n"
}

# ============================================================
# Main
# ============================================================
case "$ACTION" in
    create) create_vms ;;
    remove) remove_vms ;;
esac

# ============================================================
# Bash/Zsh completion
# ============================================================
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    # Script is run directly
    :
else
    # If sourced, enable completion
    _myvm_completions() {
        local cur="${COMP_WORDS[COMP_CWORD]}"
        local options="create remove --help"
        COMPREPLY=( $(compgen -W "$options" -- "$cur") )
    }
    complete -F _myvm_completions myvm
fi
