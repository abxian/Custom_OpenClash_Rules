# OpenClash compatibility rules

These lists are compatibility copies used by `templates/Custom_Ecommerce.ini`.
They track the corresponding `abxian/stash-override` lists while omitting rule
types that the deployed Mihomo/OpenClash core does not support:

- `USER-AGENT`
- `URL-REGEX`

The source revision is recorded in each list. Keep the remaining rule order
unchanged when refreshing a copy, then validate the generated configuration
with the target Mihomo core before updating the template reference.
