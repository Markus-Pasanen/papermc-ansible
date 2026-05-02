# PaperMC Infrastructure Deployment

# Create directory structure
mkdir -p minecraft-ansible/{templates,files}

# Copy all files above into place

# Edit vars.yml with your actual passphrase
nano minecraft-ansible/vars.yml

# Run the playbook
cd minecraft-ansible
ansible-playbook site.yml