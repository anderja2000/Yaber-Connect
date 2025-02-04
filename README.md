# Yaber-Connect
Create a solution connecting Alexa devices to Yaber smart projectors (K2s model), enabling voice-controlled functionality. Users can use Alexa commands to pause, play, skip, and control projector features seamlessly, enhancing user interaction and convenience.

## Project Structure 

smart-home-automation/
├── .gitignore                 # Contains your specified patterns + additions below
├── .env.example               # Template for environment variables
├── docker/
│   ├── homeassistant/
│   │   ├── docker-compose.yml  # Your provided HA compose file
│   │   └── config/
│   │       ├── configuration.yaml  # With your HTTP settings
│   │       ├── automations/        # Split automations
│   │       │   └── network-triggers.yaml  # Your laptop automations
│   │       ├── scripts/
│   │       │   └── media-controls.yaml  # Your scripts.yml content
│   │       └── themes/
│   └── cloudflared/
│       └── docker-compose.yml  # Your provided compose file
├── src/
│   └── alexa-integration/
│       ├── alexa_skill.py     # Your paste.txt content
│       └── requirements.txt
├── docs/
│   ├── ARCHITECTURE.md        # System diagram & data flow
│   ├── SECURITY.md           # Audit process & vulnerability reporting
│   └── DEPLOYMENT.md         # Step-by-step setup guide
├── config_examples/
│   ├── homeassistant/
│   │   └── configuration.yaml.sample  # Sanitized version
│   └── cloudflared/
│       └── config.yml.sample
├── scripts/                   # Maintenance utilities
│   ├── backup-ha.sh
│   └── renew-tunnel-cert.sh
└── tests/
    ├── unit/
    │   └── test_alexa_skill.py
    └── integration/
        └── test_ha_cloudflare.py
