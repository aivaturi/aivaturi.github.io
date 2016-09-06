yout: post
title:  "Ansible without ansible.cfg and ec2.ini"
date:   2016-09-04 21:25:00
categories: Ansible AWS
---

So you have all your infrastructure in AWS and you chose ansible to manage the said infrastructure. You played with Ansible and started to do more advanced stuff with it. And then slowly come to the realization that you have to hold on to two crutches : [ansible.cfg] [1] and [ec2.ini] [2]. But ideally, you want that to be dynamic as well - may be set all the configs via environment variables.

This is mostly possible without much modification. In [constants.py][3] of Ansible source, you can see all the environment variables, it already recognizes. This allows you to mostly get rid of ansible.cfg. Now, ec2.ini doesn't afford us any such luxury. It is used by only one script : [ec2.py][4]

## Making ec2.ini dynamic

The goal is very simple - make ""ec2.ini" dynamic. And that can be achieved by converting all the config vars in ec2.ini to ENV variables and then read them in ec2.py. It is a standalone (and rather straightforward) python script. Hacking that should be easy. But, if you don't want to touch it explicitly, you can of course subclass Ec2Inventory() in ec2.py and override read_settings() method. An example of how you would modify that method:

```python
def read_settings(self):
        ''' This method overrides the default and uses env vars '''

        # Regions
        self.regions = []
        configRegions = os.environ.get('EC2_REGION')
        configRegions_exclude = os.environ.get('EC2_INI_REGIONS_EXCLUDE',
                                               'us-gov-west-1,cn-north-1')
        if (configRegions == 'all'):
            if self.eucalyptus_host:
                self.regions.append(boto.connect_euca(host=self.eucalyptus_host).
                                    region.name, **self.credentials)
            else:
                for regionInfo in ec2.regions():
                    if regionInfo.name not in configRegions_exclude:
                        self.regions.append(regionInfo.name)
        else:
            self.regions = configRegions.split(",")

        # is eucalyptus?
        self.eucalyptus_host = None
        self.eucalyptus = False

        # Destination addresses
        self.destination_variable = os.environ.get('EC2_INI_DEST_VAR',
                                                   'private_dns_address')
        self.vpc_destination_variable = os.environ.get('EC2_INI_VPC_DEST_VAR',
                                                       'private_ip_address')

        self.hostname_variable = os.environ.get('EC2_INI_HOSTNAME_VAR', None)
        self.destination_format = os.environ.get('EC2_INI_DEST_FORMAT', None)
        self.destination_format_tags = os.environ.get('EC2_INI_DEST_FORMAT_TAGS',
                                                      None)
        if self.destination_format_tags:
            self.destination_format_tags = self.destination_format_tags.split(',')

        # Route53
        self.route53_enabled = True
        self.route53_excluded_zones = []
        if os.environ.get('EC2_INI_ROUTE53_EXCLUDED_ZONES'):
            self.route53_excluded_zones.extend(
                os.environ.get('EC2_INI_ROUTE53_EXCLUDED_ZONES').split(','))

        # Include RDS instances?
        self.rds_enabled = False
        self.include_rds_clusters = False

        # Include ElastiCache instances?
        self.elasticache_enabled = False

        # Return all EC2 instances?
        self.all_instances = False

        # Instance states to be gathered in inventory. Default is 'running'.
        # Setting 'all_instances' to 'yes' overrides this option.
        ec2_valid_instance_states = [
            'pending',
            'running',
            'shutting-down',
            'terminated',
            'stopping',
            'stopped'
        ]
        self.ec2_instance_states = ['running']

        # Return all RDS instances? (if RDS is enabled)
        self.all_rds_instances = False

        # Return all ElastiCache replication groups? (if ElastiCache is enabled)
        self.all_elasticache_replication_groups = False

        # Return all ElastiCache clusters? (if ElastiCache is enabled)
        self.all_elasticache_clusters = False

        # Return all ElastiCache nodes? (if ElastiCache is enabled)
        self.all_elasticache_nodes = False

        # boto configuration profile (prefer CLI argument)
        self.boto_profile = self.args.boto_profile

        # Cache related
        cache_dir = os.path.expanduser(os.environ.get('EC2_INI_CACHE_DIR',
                                                      '/tmp/.ansible'))
        if not os.path.exists(cache_dir):
            os.makedirs(cache_dir)

        cache_name = 'ansible-ec2'
        aws_profile = lambda: (self.boto_profile or
                               os.environ.get('AWS_PROFILE') or
                               os.environ.get('AWS_ACCESS_KEY_ID') or
                               self.credentials.get('aws_access_key_id', None))
        if aws_profile():
            cache_name = '%s-%s' % (cache_name, aws_profile())
        self.cache_path_cache = cache_dir + "/%s.cache" % cache_name
        self.cache_path_index = cache_dir + "/%s.index" % cache_name
        self.cache_max_age = 300

        self.expand_csv_tags = False

        # Configure nested groups instead of flat namespace.
        self.nested_groups = False

        # Replace dash or not in group names
        self.replace_dash_in_groups = True

        # Configure which groups should be created.
        group_by_options = [
            'group_by_instance_id',
            'group_by_region',
            'group_by_availability_zone',
            'group_by_ami_id',
            'group_by_instance_type',
            'group_by_key_pair',
            'group_by_vpc_id',
            'group_by_security_group',
            'group_by_tag_keys',
            'group_by_tag_none',
            'group_by_route53_names',
            'group_by_rds_engine',
            'group_by_rds_parameter_group',
            'group_by_elasticache_engine',
            'group_by_elasticache_cluster',
            'group_by_elasticache_parameter_group',
            'group_by_elasticache_replication_group',
        ]
        for option in group_by_options:
            setattr(self, option, True)

        # Do we need to just include hosts that match a pattern?
        self.pattern_include = None

        # Do we need to exclude hosts that match a pattern?
        self.pattern_exclude = None

        # Instance filters (see boto and EC2 API docs). Ignore invalid filters.
        self.ec2_instance_filters = defaultdict(list)
```

The above code snippet is rather simple. The only caveat being, I put sane defaults for things I don't use. For e.g. there is no need for elasticache in my personal test environment. You should go through each individual ENV var setting and modify it, if you use those services.

## Setting up the environment

Now that your dynamic inventory configuration is also dynamic, time to automate the setting up of the environment as well. I use a simple bash script for that:

```bash
# Make sure older style env vars are unset
unset ANSIBLE_CONFIG
unset EC2_INI_PATH

# Find where we are...
export ANSIBLE_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# See if we can find out the region we're in.
if [ -e "/usr/bin/ec2metadata" ]; then
    region=`/usr/bin/ec2metadata --availability-zone | grep -Po "(us|sa|eu|ap)-(north|south|central)?(east|west)?-[0-9]+"`
    if [ -z ${EC2_REGION+x} ]
    then
      export EC2_REGION="$region"
    fi
fi

# ANSIBLE default configs
export ANSIBLE_INVENTORY="$ANSIBLE_DIR/ec2_ini.py"
export ANSIBLE_LIBRARY="$ANSIBLE_DIR/lib"
export ANSIBLE_LOG_PATH="$ANSIBLE_DIR/log/ansible.log"
export ANSIBLE_NOCOWS=1
export ANSIBLE_REMOTE_TEMP="/tmp"
export ANSIBLE_ROLES_PATH="$ANSIBLE_DIR/roles"
export ANSIBLE_SSH_ARGS="-C -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -o ControlMaster=auto -o ControlPersist=60s"
export ANSIBLE_REMOTE_PORT=22
export ANSIBLE_PRIVATE_KEY_FILE="$HOME/.ssh/aws_ssh_key.pem"

# Plugins path
export ANSIBLE_ACTION_PLUGINS="$ANSIBLE_DIR/plugins/action:/usr/share/ansible_plugins/action_plugins"
export ANSIBLE_CACHE_PLUGINS="$ANSIBLE_DIR/plugins/cache:/usr/share/ansible_plugins/cache_plugins"
export ANSIBLE_CALLBACK_PLUGINS="$ANSIBLE_DIR/plugins/callback:/usr/share/ansible_plugins/callback_plugins"
export ANSIBLE_CONNECTION_PLUGINS="$ANSIBLE_DIR/plugins/connection:/usr/share/ansible_plugins/connection_plugins"
export ANSIBLE_FILTER_PLUGINS="$ANSIBLE_DIR/plugins/filter:/usr/share/ansible_plugins/filter_plugins"
export ANSIBLE_LOOKUP_PLUGINS="$ANSIBLE_DIR/plugins/lookup:/usr/share/ansible_plugins/lookup_plugins"
export ANSIBLE_SHELL_PLUGINS="$ANSIBLE_DIR/plugins/shell:/usr/share/ansible_plugins/shell_plugins"
export ANSIBLE_STRATEGY_PLUGINS="$ANSIBLE_DIR/plugins/strategy:/usr/share/ansible_plugins/strategy_plugins"
export ANSIBLE_VARS_PLUGINS="$ANSIBLE_DIR/plugins/vars:/usr/share/ansible_plugins/vars_plugins"

eval "cat <<EOF
$(<${ANSIBLE_DIR}/all.tmpl)
EOF
" > "$ANSIBLE_DIR/group_vars/all"

```

Simply source the script in a bash shell.

In the above script, we do some extra steps to make our lives simpler. Since your ansible host is deployed in AWS, most of the instances will come with a mechanism to detect what region that instance is in. In Ubuntu, you can find /usr/bin/ec2metadata. This helps us set the EC2_REGION env var, which will be used in our read_settings() method snippet above.

Apart from that there is one extra step I do. Setting up "all" group_vars file. This typically contains variables that apply to your entire environment. The last 4 lines render that file in group_vars folder. Here is an example of all.tmpl:

```bash
ssh_remote_port:  ${ANSIBLE_REMOTE_PORT}
ansible_ssh_private_key_file: "${ANSIBLE_PRIVATE_KEY_FILE}"
ansible_dir: "${ANSIBLE_DIR}"
```

You can of course add other global vars in that template.

We have done the following so far:

- Got rid of ansible.cfg
- Got rid of ec2.ini
- Wrote a simple "setenv" script to set up the ansible environment in a single step.

## Consul and Vault

At this point, you might wonder about two things I glossed over:

1. What about the values for ENV vars themselves? Don't we have to hardcode them? How is it going to be different from ansible.cfg or ec2.ini?
2. What about AWS API keys?

For (1), there is indeed a decent solution : [Consul][5]. The "setenv" script should be modified to read all those values from consul. Here is an example:

```bash
# assuming keys in consul are : v1/kv/settings/ansible/inventory,
# v1/kv/settings/ansible/lib and so on...

export KEY_ROOT="settings"
export CONSUL_SERVER="consul.example.com"

consul-export() {
    # Remove any temp env files
    rm -f /tmp/env_vars

    ################################
    # Gather ANSIBLE env vars
    ################################
    declare key="${KEY_ROOT}/ansible"
    curl --fail -Ss "http://$CONSUL_SERVER:8500/v1/kv/$key/?recurse" \
        | python -c "$(printf %b 'import json,sys,os,re\nobj=json.load(sys.stdin)\nfor item in obj: print("%s %s" % (re.sub(os.environ["KEY_ROOT"]+"/","", item["Key"]), item["Value"]))')" \
        | sed "s/\//_/g" \
        | grep -v '^\s*$' \
        | \
        while read key value; do
            key=$(echo "$key" | tr 'a-z' 'A-Z')
            key="${key//\//_}"
            printf "export $key=\"%s\"\n" "$(echo "$value" | base64 -d)" >> /tmp/env_vars
        done

    # Export all the vars
    source "/tmp/env_vars"
}
```

The above snippet makes our "setenv" truly dynamic. All settings can be controlled from Consul and if you need to change any values, change it there and run your ansible command. As long as your playbooks are designed for it, they will simply work.

For the second question - the solution is [Hashicorp Vault][6] (not to be confused with Ansible Vault), specifically: [AWS Secret Backend][7]. Make sure that your lease_duration is small enough so that the keys are automatically revoked by Vault sooner than later. This also has a really good side effect of you being conscious about security.

[1]: https://github.com/ansible/ansible/blob/devel/examples/ansible.cfg
[2]: https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini
[3]: https://github.com/ansible/ansible/blob/devel/lib/ansible/constants.py#L153
[4]: https://github.com/ansible/ansible/blob/devel/contrib/inventory/ec2.py#L216
[5]: https://consul.io
[6]: https://vaultproject.io
[7]: https://www.vaultproject.io/docs/secrets/aws/index.html
