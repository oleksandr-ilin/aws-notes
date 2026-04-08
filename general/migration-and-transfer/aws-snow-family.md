Overview
AWS Snow Family (Snowcone, Snowball, Snowmobile) provides physical devices for large-scale offline data transfer to AWS.

Tasks
- Order device, copy data locally to device, ship to AWS for ingestion, verify imports.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Online transfer (DataSync) | No logistics | Very slow for PB-scale |
| Third-party shipping services | Flexibility | Integration work

Limitations
- Logistics, shipping time, and device capacity planning required.

Price info
- Device rental fees and import/export data transfer charges apply.

Network & multi-region
- Devices are regional/edge; plan target region and import timelines.

Popular use cases
- Petabyte-scale data migrations, large media archives transfer.

When Not to Use
- Small datasets or when low-latency incremental syncs are required.

DR strategy
- Snow devices can seed DR copies in another region when network transfer is impractical.
