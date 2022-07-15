---
layout: post
title:  "Create Solana Validator RPC Only Node"
date:   2022-07-15 06:58:15 +0700
categories: solana
---
# Disk Setup

{% highlight terminal %}
mkdir -p /solana/{ledger,account,spare}
{% endhighlight %}

Check if the directory is okay by running the ls command on `/solana/` directory. Also, make sure all the directories read, write and access permissions is okay by running the chmod on the created directories.
{% highlight terminal %}
chmod a+w /solana/ledger
chmod a+w /solana/account
chmod a+w /solana/spare
{% endhighlight %}

# Setup the Solana Validator

Before you start, make sure you install the solana cli first which you can find the information on how to install the latest package [here](https://docs.solana.com/cli/install-solana-cli-tools). Reboot your computer to make sure everything is going okay.

On this config we will connect directly as a validator (non-voting) for Solana mainnet-beta. First configure the Solana tools that you installed.
{% highlight terminal %}
solana config set --url https://api.mainnet-beta.solana.com
{% endhighlight %}

Then run the sys-tuner for one time, this is to configure your computers internals to the recommended setup.
{% highlight terminal %}
$(command -v solana-sys-tuner) --user $(whoami) > sys-tuner.log 2>&1 &
{% endhighlight %}

This will run one-time, you need to run it after you reboot your system. There is also another way, on which you need to configure a `systemd` service. To configure a `systemd` service, create a file named `solana-sys-tuner.service` in the directory `/etc/systemd/system`.
{% highlight terminal %}
cat > /etc/systemd/system/solana-sys-tuner.service << EOF
[Unit]
Description=Solana System Tuner
After=network.target
Before=sol.service

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/root/solana-sys-tuner.sh

[Install]
WantedBy=multi-user.target
WantedBy=sol.service
EOF
{% endhighlight %}

This will create the service and now you can run `systemctl enable --now solana-sys-tuner.service` to enable it at boot and start the service right now. You can also do the manual way without having to run the `solana-sys-tuner` binary, just follow this tutorial from the official documentation [here](https://docs.solana.com/running-validator/validator-start#manual).
Also don’t forget to create the `solana-sys-tuner.sh` on your user home root directory.
{% highlight terminal %}
cat > ~/solana-sys-tuner.sh << EOF
#!/usr/bin/env bash
set -ex

exec /root/.local/share/solana/install/active_release/bin/solana-sys-tuner --user root
EOF
{% endhighlight %}

Now you can now start the validator, to start prepare first a validator keypair.
{% highlight terminal %}
solana-keygen new -o ~/validator-keypair.json
{% endhighlight %}

This will create a validator keypair at your user home directory. Don’t forget to save the output generated BIP39 seedphrase. **DON’T FORGET**. Once done, if you forgot your public key, you can view it using the command `solana-keygen pubkey ~/validator-keypair.json`. You will need the public key for later commands.

Set the validator keypair in your Solana cli tool:
{% highlight terminal %}
solana config set --keypair ~/validator-keypair.json
{% endhighlight %}

That’s all for configuration, we can now start the validator. Create an simple shell script to contain the run parameters of the `solana-validator` command, so it will be easier to modify and adjust later on.
{% highlight terminal %}
cat > validator.sh << EOF
#!/usr/bin/env bash

set -e

exec solana-validator \
    --no-voting \
    --identity ~/validator-keypair.json \
    --known-validator 7Np41oeYqPefeNQEHSv1UDhYrehxin3NStELsSKCT4K2 \
    --known-validator GdnSyH3YtwcxFvQrVVJMm1JhTS4QVX7MFsX56uJLUfiZ \
    --known-validator DE1bawNcRJB9rVm3buyMVfr8mBEoyyu73NBovf2oXJsJ \
    --known-validator CakcnaRDHka2gXyfbEd2d3xsvkJkqsLw2akB3zsN1D2S \
    --only-known-rpc \
    --ledger /solana/ledger \
    --accounts /solana/account \
    --rpc-port 8899 \
    --rpc-bind-address 0.0.0.0 \
    --dynamic-port-range 8000-8100 \
    --entrypoint entrypoint.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint2.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint3.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint4.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint5.mainnet-beta.solana.com:8001 \
    --expected-genesis-hash 5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d \
    --wal-recovery-mode skip_any_corrupted_record \
    --limit-ledger-size \
    --no-port-check \
    --enable-rpc-transaction-history \
    --full-rpc-api \
    --log /solana/spare/logs/solana-validator.log
EOF
{% endhighlight %}

This validator flags specify that RPC is open to public, and its only rpc-mode due to the `--no-voting flag`. The flags also specify that the RPC transaction history is enabled which will make the ledger disk be big, check the flag `--enable-rpc-transaction-history`. Read every flags from the `solana-validator` binary by executing the `--help` flag.

Wait for a while, it will download a very big snapshot so you could catch up at the latest transactions. The ledger will only contain all the latest transaction history of the Solana chain. This will take time depending on the speed of your machine and speed of network. Once you see there are no percentage or anything, then its good to go.

Make sure its on the list of validator nodes using the solana gossip command.
{% highlight terminal %}
solana gossip | grep <pubkey>
{% endhighlight %}

That’s all, now you’re part of the validators. Another thing, in order to run it on reboot add the `systemd` service file, create the file using the command below on same directory as the `solana-sys-tuner.service`.
{% highlight terminal %}
cat > /etc/systemd/system/sol.service << EOF
[Unit]
Description=Solana Validator
After=network.target
Wants=solana-sys-tuner.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
Environment="PATH=/bin:/usr/bin:/root/.local/share/solana/install/active_release/bin"
ExecStart=/root/validator.sh

[Install]
WantedBy=multi-user.target
EOF
{% endhighlight %}

Then enable it at boot using the command `systemctl enable --now sol.service`. Make sure that it doesn’t have errors by checking the service status. Last thing to mention regarding logs as it can become large quickly, make sure to create a logrotate rule, the command below which I grabbed from the official documentation.
{% highlight terminal %}
cat > logrotate.sol <<EOF
/root/solana-validator.log {
  rotate 7
  daily
  missingok
  postrotate
    systemctl kill -s USR1 sol.service
  endscript
}
EOF

sudo cp logrotate.sol /etc/logrotate.d/sol
systemctl restart logrotate.service
{% endhighlight %}
That’s all, reboot and celebrate 🎉! Don’t forget to share and leave a comment if you like this kind of articles.