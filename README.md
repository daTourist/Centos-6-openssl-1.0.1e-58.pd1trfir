### Centos 6 openssl-1.0.1e Patch - _Always_ Set "Trusted First" CLI Flag
#### Clean & simple solution by _pajkd_ @ https://community.letsencrypt.org/u/pajkd
https://community.letsencrypt.org/t/rhel-centos-6-openssl-client-compatibility-after-dst-root-ca-x3-expiration/161032/62

_This solution will take the latest openssl source RPM packages and will add a simple patch to always enable the option "trusted first"_

##### 0. Pre-update packages to latest version available
```
sudo yum update ca-certificates nss openssl -y
```

##### 1. Confirm Issue/Bug Exists & Patch Applicable
```
openssl s_client -connect community.letsencrypt.org:https
> Verify return code: 20 (unable to get local issuer certificate)

openssl s_client -trusted_first -connect community.letsencrypt.org:https
> Verify return code: 0 (ok)
```

##### 2. Install rpm-build / yum-utils / krb5-devel
```
sudo yum install rpm-build yum-utils krb5-devel -y
```

##### 3. Prepare rpm-build ***USER*** Environment // *DO NOT RUN AS ROOT*
```
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
```

##### 4. Download _openssl_ source RPM using yumdownloader _OR_ Other Source // *DO NOT RUN AS ROOT*
```
yumdownloader --source openssl

 - OR -

wget -D ~/ http://bay.uchicago.edu/centos-vault/centos/6/updates/Source/SPackages/openssl-1.0.1e-58.el6_10.src.rpm
rpm -i ~/openssl-1.0.1e-58.el6_10.src.rpm
```

##### 5. _Optionally_ build the original OpenSSL RPM Files // *DO NOT RUN AS ROOT*
```
rpmbuild -bp ~/rpmbuild/SPECS/openssl.spec
rpmbuild -ba ~/rpmbuild/SPECS/openssl.spec
ls -la ~/rpmbuild/RPMS/x86_64/
```

##### 6a. Create the patch "always_trusted_first" based on _pajkd_ // *DO NOT RUN AS ROOT*
```
cat <<EOF > ~/rpmbuild/SOURCES/openssl-1.0.1e-always_trusted_first.patch
--- openssl-1.0.1e/crypto/x509/x509_vfy.c.before_trusted_first_always/*TAB*/2021-10-02 20:36:02.613066113 +0200
+++ openssl-1.0.1e/crypto/x509/x509_vfy.c/*TAB*/2021-10-02 20:37:06.890422274 +0200
@@ -206,7 +206,7 @@
 /*TAB*//*TAB*//* If we are self signed, we break */
 /*TAB*//*TAB*/if (ctx->check_issued(ctx, x,x)) break;
 /*TAB*//*TAB*//* If asked see if we can find issuer in trusted store first */
-/*TAB*//*TAB*/if (ctx->param->flags & X509_V_FLAG_TRUSTED_FIRST)
+/*TAB*//*TAB*/if (/* PATCH TRUSTED FIRST ALWAYS - ctx->param->flags & */ X509_V_FLAG_TRUSTED_FIRST)
 /*TAB*//*TAB*//*TAB*/{
 /*TAB*//*TAB*//*TAB*/ok = ctx->get_issuer(&xtmp, ctx, x);
 /*TAB*//*TAB*//*TAB*/if (ok < 0)
EOF

sed -i "s/\/\*TAB\*\//\t/g" ~/rpmbuild/SOURCES/openssl-1.0.1e-always_trusted_first.patch
cat ~/rpmbuild/SOURCES/openssl-1.0.1e-always_trusted_first.patch
```

##### 6b. _Alternatively_ download the patch "always_trusted_first" by _pajkd_ // *DO NOT RUN AS ROOT*
```
wget --no-check-certificate -D ~/rpmbuild/SOURCES/ https://unix.tpc.cz/anonftp/servis/openssl-centos6-LE-2021/openssl-1.0.1e-always_trusted_first.patch
cat ~/rpmbuild/SOURCES/openssl-1.0.1e-always_trusted_first.patch
```

##### 7. Update release & add patch to OpenSSL RPM Specs File // *DO NOT RUN AS ROOT*
```
sed -i "s/Release: 58%{?dist}/Release: 58.pd1trfir%{?dist}/g" ~/rpmbuild/SPECS/openssl.spec
sed -i "s/Patch168: openssl-1.0.1e-cve-2019-1559.patch/Patch168: openssl-1.0.1e-cve-2019-1559.patch\nPatch169: openssl-1.0.1e-always_trusted_first.patch/g" ~/rpmbuild/SPECS/openssl.spec
sed -i "s/%patch168 -p1 -b .error-state/%patch168 -p1 -b .error-state\n%patch169 -p1 -b .always_trfirst/g" ~/rpmbuild/SPECS/openssl.spec
grep atch169 ~/rpmbuild/SPECS/openssl.spec
```

##### 8. Build patched OpenSSL RPM Files // *DO NOT RUN AS ROOT*
```
rpmbuild -ba ~/rpmbuild/SPECS/openssl.spec
ls -la ~/rpmbuild/RPMS/x86_64/
```

##### 9. List original & patched OpenSSL RPM Files
```
ls -la ~/rpmbuild/RPMS/x86_64/
> openssl-1.0.1e-58.el6.x86_64.rpm
> openssl-1.0.1e-58.pd1trfir.el6.x86_64.rpm
> openssl-debuginfo-1.0.1e-58.el6.x86_64.rpm
> openssl-debuginfo-1.0.1e-58.pd1trfir.el6.x86_64.rpm
> openssl-devel-1.0.1e-58.el6.x86_64.rpm
> openssl-devel-1.0.1e-58.pd1trfir.el6.x86_64.rpm
> openssl-perl-1.0.1e-58.el6.x86_64.rpm
> openssl-perl-1.0.1e-58.pd1trfir.el6.x86_64.rpm
> openssl-static-1.0.1e-58.el6.x86_64.rpm
> openssl-static-1.0.1e-58.pd1trfir.el6.x86_64.rpm
```

##### 10. Verify installed OpenSSL RPM Files
```
sudo rpm -qa | sort | grep openssl
> openssl-1.0.1e-58.el6_10.x86_64
> openssl-devel-1.0.1e-58.el6_10.x86_64
```

##### 11. Install patched OpenSSL RPM File(s)
_Based on step 10, you need to specify **all** OpenSSL installed RPM files to be replaced with the patched version_
```
sudo rpm -Uhv ~/rpmbuild/RPMS/x86_64/openssl-1.0.1e-58.pd1trfir.el6.x86_64.rpm ~/rpmbuild/RPMS/x86_64/openssl-devel-1.0.1e-58.pd1trfir.el6.x86_64.rpm
```

##### 12. Verify installed patched OpenSSL RPM Files
```
sudo rpm -qa | sort | grep openssl
> openssl-1.0.1e-58.pd1trfir.el6.x86_64
> openssl-devel-1.0.1e-58.pd1trfir.el6.x86_64.rpm
```

##### 13. Confirm Issue/Bug Patched
```
openssl s_client -connect community.letsencrypt.org:https
> Verify return code: 0 (ok)
```

##### 14. _Reverting Changes:_ Re-Install original OpenSSL RPM File(s)
```
sudo rpm -Uhv --oldpackage ~/rpmbuild/RPMS/x86_64/openssl-1.0.1e-58.el6_10.x86_64.rpm ~/rpmbuild/RPMS/x86_64/openssl-devel-1.0.1e-58.el6_10.x86_64.rpm
sudo rpm -qa | sort | grep openssl
> openssl-1.0.1e-58.el6_10.x86_64
> openssl-devel-1.0.1e-58.el6_10.x86_64
```
