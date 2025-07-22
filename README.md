#!/bin/bash
set -e

# Define paths
USER_HOME=$(eval echo ~$USER)
PKI_BASE="$USER_HOME/pki"
ROOTCA_DIR="$PKI_BASE/ca"
SUBCA_DIR="$PKI_BASE/subca"
USER_DIR="$PKI_BASE/user"

echo "📦 Installing required packages..."
sudo apt update
sudo apt install -y tree openssl wget

echo "📁 Setting up Root CA..."
mkdir -p "$ROOTCA_DIR"/{certs,crl,newcerts,private,subca/csr,subca/certs}
cd "$ROOTCA_DIR"
touch index.txt index.txt.attr
echo 1000 > serial
echo 1000 > crlnumber

echo "🔐 Creating Root CA private key..."
openssl genrsa -aes256 -out private/ca.key.pem 4096
chmod 400 private/ca.key.pem

echo "🌐 Downloading and fixing rootca.cnf..."
wget -q -O rootca.cnf http://192.168.1.251/sw/pki/rootca.cnf
sed -i "s|^dir.*|dir = $ROOTCA_DIR|g" rootca.cnf
sed -i "s|^private_key.*|private_key = \$dir/private/ca.key.pem|g" rootca.cnf
sed -i "s|^certificate.*|certificate = \$dir/certs/ca.cert.pem|g" rootca.cnf

echo "📜 Generating Root CA certificate..."
openssl req -config rootca.cnf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem

echo "📁 Setting up Sub CA..."
mkdir -p "$SUBCA_DIR"/{certs,crl,newcerts,private,csr}
cd "$SUBCA_DIR"
touch index.txt index.txt.attr
echo 1000 > serial
echo 1000 > crlnumber

echo "🔐 Creating Sub CA private key..."
openssl genrsa -aes256 -out private/subca.key.pem 4096
chmod 400 private/subca.key.pem

echo "🌐 Downloading and fixing subca.cnf..."
wget -q -O "$ROOTCA_DIR/subca.cnf" http://192.168.1.251/sw/pki/subca.cnf
sed -i "s|^dir.*|dir = $SUBCA_DIR|g" "$ROOTCA_DIR/subca.cnf"
sed -i "s|^private_key.*|private_key = \$dir/private/subca.key.pem|g" "$ROOTCA_DIR/subca.cnf"
sed -i "s|^certificate.*|certificate = \$dir/certs/subca.cert.pem|g" "$ROOTCA_DIR/subca.cnf"

echo "📜 Creating Sub CA CSR..."
openssl req -config "$ROOTCA_DIR/subca.cnf" -new -sha256 -key private/subca.key.pem -out csr/subca.csr.pem

echo "📄 Signing Sub CA with Root CA..."
cp csr/subca.csr.pem "$ROOTCA_DIR/subca/csr/"
cd "$ROOTCA_DIR"
openssl ca -config rootca.cnf -extensions v3_intermediate_ca -days 7300 -notext -md sha256 \
  -in subca/csr/subca.csr.pem -out subca/certs/subca.cert.pem

cp subca/certs/subca.cert.pem "$SUBCA_DIR/certs/"

echo "👤 Creating user cert (shuhari.local)..."
mkdir -p "$USER_DIR"
cd "$USER_DIR"
openssl genrsa -out shuhari.local.key.pem 2048
chmod 400 shuhari.local.key.pem
openssl req -config "$ROOTCA_DIR/subca.cnf" -key shuhari.local.key.pem -new -sha256 -out shuhari.local.csr.pem

cp shuhari.local.csr.pem "$SUBCA_DIR/csr/"

echo "📜 Signing user cert with Sub CA..."
cd "$SUBCA_DIR"
openssl ca -config "$ROOTCA_DIR/subca.cnf" -extensions server_cert -days 7300 -notext -md sha256 \
  -in csr/shuhari.local.csr.pem -out certs/shuhari.local.cert.pem

cp certs/shuhari.local.cert.pem "$USER_DIR/"
cp certs/subca.cert.pem "$USER_DIR/"
cp "$ROOTCA_DIR/certs/ca.cert.pem" "$USER_DIR/"

cd "$USER_DIR"
cat shuhari.local.cert.pem subca.cert.pem ca.cert.pem > shuhari.local.cert.chain.pem

echo "✅ DONE! All your final files are in:"
echo "$USER_DIR"
tree "$USER_DIR"
