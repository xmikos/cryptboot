#!/bin/bash

# Check if user is root
if [[ $UID -ne 0 ]]; then
  echo "Permission denied (you must be root)"
  exit 1
fi

# Default config
SRCDIR="$(dirname "$(readlink -f "$0")")"
EFI_KEYS_DIR="/boot/efikeys"

# Load config file
if [[ -f "/etc/cryptboot.conf" ]]; then
  source "/etc/cryptboot.conf"
elif [[ -f "$SRCDIR/cryptboot.conf" ]]; then
  source "$SRCDIR/cryptboot.conf"
fi

# Create UEFI Secure Boot keys directory (if it doesn't already exists)
mkdir -p "$EFI_KEYS_DIR"

# Main program
case "$1" in
  create)
    cd "$EFI_KEYS_DIR"

    if [[ -f KEK.esl ]] || [[ -f db.esl  ]] || [[ -f PK.auth  ]]; then
        echo "UEFI Secure Boot keys in '$EFI_KEYS_DIR' already exists!"
        read -n1 -r -s -p "Do you want to delete them and generate new ones [y/N]? " key
        echo
        if [[ "$key" != "y" ]] && [[ "$key" != "Y" ]]; then
          exit 1
        fi
    fi

    echo "Creating new UEFI Secure Boot keys..."
    read -p "Enter a Common Name: " NAME
    echo "$NAME" > NAME

    openssl req -new -x509 -newkey rsa:2048 -subj "/CN=$NAME PK/" -keyout PK.key \
                -out PK.crt -days 3650 -nodes -sha256
    openssl req -new -x509 -newkey rsa:2048 -subj "/CN=$NAME KEK/" -keyout KEK.key \
                -out KEK.crt -days 3650 -nodes -sha256
    openssl req -new -x509 -newkey rsa:2048 -subj "/CN=$NAME db/" -keyout db.key \
                -out db.crt -days 3650 -nodes -sha256

    openssl x509 -in PK.crt -out PK.cer -outform DER
    openssl x509 -in KEK.crt -out KEK.cer -outform DER
    openssl x509 -in db.crt -out db.cer -outform DER

    GUID="$(uuidgen --random)"
    echo "$GUID" > GUID

    cert-to-efi-sig-list -g $GUID PK.crt PK.esl
    cert-to-efi-sig-list -g $GUID KEK.crt KEK.esl
    cert-to-efi-sig-list -g $GUID db.crt db.esl

    echo -n > PK_null.esl

    sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.auth
    sign-efi-sig-list -k PK.key -c PK.crt PK PK_null.esl PK_null.auth

    chmod 0400 *.{key,auth}
    sync

    echo "New UEFI Secure Boot keys created in '$EFI_KEYS_DIR'"
    ;;
  enroll)
    cd "$EFI_KEYS_DIR"

    if ! [[ -f KEK.esl ]] || ! [[ -f db.esl  ]] || ! [[ -f PK.auth  ]]; then
        echo "There are no UEFI Secure Boot keys in '$EFI_KEYS_DIR'!"
        echo "Please generate them first with '$0 create'"
        exit 1
    fi

    echo "Enrolling UEFI Secure Boot KEK key..."
    efi-updatevar -e -f KEK.esl KEK

    echo "Enrolling UEFI Secure Boot db key..."
    efi-updatevar -e -f db.esl db

    echo "Enrolling UEFI Secure Boot PK key..."
    efi-updatevar -f PK.auth PK

    echo "UEFI Secure Boot keys in '$EFI_KEYS_DIR' enrolled to UEFI firmware."
    echo "You should now sign your boot loader and reboot!"
    ;;
  sign)
    cd "$EFI_KEYS_DIR"

    filename="$2"
    if [[ -z "$filename" ]]; then
      echo "You have to specify EFI boot image file for signing it with UEFI Secure Boot keys!"
      exit 1
    fi

    if ! [[ -f db.key ]] || ! [[ -f db.crt  ]]; then
        echo "There are no UEFI Secure Boot keys in '$EFI_KEYS_DIR'!"
        echo "Please generate them first with '$0 create' and then enroll them with '$0 enroll'"
        exit 1
    fi

    echo "Signing file '$filename' with UEFI Secure Boot keys..."
    sbsign --key db.key --cert db.crt --output "$filename" "$filename"
    sync
    ;;
  verify)
    cd "$EFI_KEYS_DIR"

    filename="$2"
    if [[ -z "$filename" ]]; then
      echo "You have to specify EFI boot image file to verify its signature!"
      exit 1
    fi

    if ! [[ -f db.crt  ]]; then
        echo "There are no UEFI Secure Boot keys in '$EFI_KEYS_DIR'!"
        echo "Please generate them first with '$0 create' and then enroll them with '$0 enroll'"
        exit 1
    fi
    echo "List of all signatures in '$filename':"
    sbverify --list "$filename"
    echo

    echo "Verifying signature with UEFI Secure Boot keys..."
    sbverify --cert db.crt "$filename"
    ;;
  list)
    echo "All UEFI Secure Boot keys enrolled in your UEFI firmware:"
    efi-readvar
    ;;
  status)
    echo -n "UEFI Secure Boot status: "
    secureboot="$(od -An -t u1 --read-bytes=1 --skip-bytes=4 /sys/firmware/efi/efivars/SecureBoot-*)"
    if [[ "$secureboot" -eq 1 ]]; then
        echo "ACTIVE"
        exit 0
    else
        echo "INACTIVE"
        exit 1
    fi
    ;;
  *)
    echo "Usage: $(basename "$0") {create,enroll,sign,verify,list} [file-to-sign-or-verify]"
    echo
    echo "Manage UEFI Secure Boot keys"
    echo
    echo "Commands:"
    echo "    create  Generate new UEFI Secure Boot keys"
    echo "    enroll  Enroll new UEFI Secure Boot keys to your UEFI firmware"
    echo "            (you have to clear old keys in your UEFI firmware setup utility first)"
    echo "    sign    Sign EFI boot image file with your UEFI Secure Boot keys"
    echo "    verify  Verify signature of EFI boot image file with your UEFI Secure Boot keys"
    echo "    list    List all UEFI Secure Boot keys enrolled in your UEFI firmware"
    echo "    status  Check if UEFI Secure Boot is active or inactive"
esac
