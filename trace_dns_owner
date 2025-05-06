#!/bin/bash
## Usage: Use this script with your organization owned public domain example - ./trace_dns_owner.sh aa.bb.example.net

var2="--profile <your aws profile>"
var3='--output json'
outputfile="dns-details.csv"
wrap_text() {
    local text="$1"
    local max_width="$2"
    echo "$text" | fold -s -w "$max_width" | awk '{printf "%s\n", $0}'
}
#aws sso login $var2
hostname=$1

# Define the input string
input_string="$hostname"

# Find the position of the third period from the end (Converting hostname into queryable string)
third_period_pos=$(echo "$input_string" | awk -F '.' '{print NF-2}')

# Remove everything before the third period
result=$(echo "$input_string" | cut -d '.' -f "$third_period_pos"-)
hostedZone="$result."
result1=$(aws configservice select-aggregate-resource-config --configuration-aggregator-name "aws-controltower-GuardrailsComplianceAggregator" --expression "SELECT * WHERE resourceName = '$hostedZone'" --profile <your aws profile> --output text)
account_id=$(echo "$result1" | sed -n 's/.*"accountId":\("\([^"]*\)"\).*/\2/p')
hostedzoneid=$(echo "$result1" | sed -n 's/.*"resourceId":\("\([^"]*\)"\).*/\2/p')
awsRegion=$(echo "$result1" | sed -n 's/.*"awsRegion":\("\([^"]*\)"\).*/\2/p')
var1="--profile <your aws profile>-$account_id"


# Perform DNS lookup to get public IP address
pubIpAddress=$(dig +short $hostname)
echo ""
echo ""
printf "%-20s\n" "AWS Details and Public IP Address for $hostname"
printf "%-20s\n" "-------------------*-------------------*-------------------*-------------------"
printf "%-20s\n" "$pubIpAddress"
echo ""
printf "%-20s\n" "Account ID that contains above domain hosted zone $account_id"
echo ""
printf "%-20s\n" "The hostedzone ID is $hostedzoneid"
echo ""
printf "%-20s\n" "The region is $awsRegion"
echo ""
printf "%-20s\n" "Checking load balancer details for above IP address"
echo ""

counter=0

# Loop through each public IP address and A records
for pubIp in $pubIpAddress; do

# Checks if the A record contains cloudfront distribution and perform Config query on cloudfront to get tags
    if [[ $pubIp == *"cloudfront.net"* ]]; then
        distributionId="${pubIp%.}"
        printf "%-20s\n" "Its a cloudfront distribution with $distributionId, Adding additional details in csv file"
        aws configservice select-aggregate-resource-config --configuration-aggregator-name "aws-controltower-GuardrailsComplianceAggregator" \
        --expression "SELECT configuration.domainName, accountId, tags WHERE resourceType = 'AWS::CloudFront::Distribution' AND configuration.domainName LIKE '$distributionId%'" \
        $var2 $var3 | jq -r --arg additional_column_value "$hostname" \
        '.Results[] | fromjson | [$additional_column_value, .configuration.domainName, .accountId, (.tags | tostring)] | @csv' >> "$outputfile"
    else

# Retrieve load balancer description of IP addresses using AWS Config
        lbdesc=$(aws configservice select-aggregate-resource-config --configuration-aggregator-name "aws-controltower-GuardrailsComplianceAggregator" \
        --expression "SELECT resourceId, configuration.description WHERE configuration.association.publicIp = '$pubIp'" \
        $var2 $var3 | jq -r '.Results[] | fromjson | .configuration.description')
    
        if [ -z "$lbdesc" ]; then
            echo "Load balancer not found for $pubIp, searching the next IP in list "
            counter=$((counter + 1))
        else
# Extract load balancer name
            lbname=$(echo $lbdesc | awk -F'/' '{print $2}')
            printf "\n%-20s\n" "Load Balancer Name"
            printf "%-20s\n" "-------------------"
            printf "%-20s\n" "$lbname"

            echo "Saving Loadbalancer details in CSV"

            # Retrieve load balancer details using AWS Config and saving in output file
            aws configservice select-aggregate-resource-config --configuration-aggregator-name "aws-controltower-GuardrailsComplianceAggregator" \
            --expression "SELECT resourceId, resourceName, relationships.resourceId, configuration.dNSName, configuration.canonicalHostedZoneId, tags WHERE resourceType = 'AWS::ElasticLoadBalancingV2::LoadBalancer' AND resourceName LIKE '$lbname'" \
            $var2 $var3 | jq -r --arg additional_column_value "$hostname" \
            '.Results[] | fromjson | [$additional_column_value, .resourceId, .resourceName, .configuration.dNSName, .configuration.canonicalHostedZoneId, (.tags | tostring)] | @csv' >> "$outputfile"
        fi
    fi
done
echo ""
echo ""
echo "Couldn't find load balancer for $counter IP address, Checking DNS Route53"

# Check if counter is not equal to zero and do a DNS search ( API Call)
if [ "$counter" -ne 0 ]; then
    DNS_RECORDS=$(aws route53 list-resource-record-sets --hosted-zone-id $hostedzoneid --query "ResourceRecordSets[?Name == '$hostname.' && Type == 'A'].AliasTarget.DNSName" $var1 --output text | sed 's/\.$//')
    if [ -z "$DNS_RECORDS" ]; then
        DNS_RECORDS=$(aws route53 list-resource-record-sets --hosted-zone-id $hostedzoneid --query "ResourceRecordSets[?Name == '$hostname.' && Type == 'CNAME'].ResourceRecords[0].Value" $var1 --output text | sed 's/\.$//')
    fi
fi
echo ""
printf "%-20s\n" "DNS Record is $DNS_RECORDS"
echo ""
echo ""
