# EcoBarter 2.0 - USSD Platform

## What It Does
EcoBarter 2.0 is a USSD-based waste management platform that makes recycling accessible to all Nigerians, regardless of smartphone ownership or internet access. Users can dial a simple code from any mobile phone to request waste pickups, check recyclable prices, manage their rewards wallet, and earn money by recycling - all without downloading an app or using data.

## Why It Matters
Nigeria's waste management sector is dominated by the informal economy, yet existing digital solutions like the original EcoBarter app remain invisible to most Nigerians. With only 45% of Nigerians owning smartphones and average 3G/4G availability at just 65%, app-based solutions exclude the majority of potential users - especially waste agents and everyday citizens who use feature phones.

EcoBarter 2.0 solves the "unknownness" problem by meeting people where they already are: on basic mobile phones using USSD technology that works offline, loads instantly, and requires no downloads.

## Who It's For
- **Waste collectors and agents** using feature phones without internet access
- **Everyday Nigerians** who want to earn money from recyclables but lack smartphones
- **Communities** in areas with poor network connectivity
- **Environmental organisations** seeking inclusive recycling solutions

## Key Features
- **Check Pricelist**: View current rates for plastic, metal, paper, and other recyclables
- **Request Pickup**: Schedule waste collection at your location
- **Rewards Wallet**: Withdraw cash or buy airtime with earned points
- **Request Drop-off Point**: Find nearby collection centers
- **Pickup Status**: Track your scheduled collections
- **Points Tracker**: Monitor earnings from recycling activities

## Technologies Used
- **Platform**: USSD (Unstructured Supplementary Service Data)
- **Backend**: Node.js with Express
- **Database**: MongoDB
- **USSD Gateway**: Africa's Talking API
- **Session Management**: Redis
- **SMS Notifications**: Africa's Talking SMS Gateway
- **Hosting**: Railway/Heroku

## The Problem We're Solving
The original EcoBarter app offers a superior waste management solution but remains practically invisible because:
- Most target users don't own smartphones
- Network instability makes app usage frustrating
- Data costs create barriers to regular usage
- The informal waste sector operates outside digital platforms

EcoBarter 2.0 transforms this unknown product into an accessible solution that works within Nigeria's infrastructure constraints.

## Success Metrics
Our goal is to achieve **5,000 active USSD users within 3 months** of launch, proving that accessibility drives adoption in Nigeria's waste management sector.

## Technical Documentation
For detailed system architecture, data models, and technical implementation, see [systemArchitecture.md](./systemArchitecture.md).

## User Workflow
View the complete USSD user journey: [EcoBarter USSD Workflow on Figma](https://www.figma.com/board/x2Qn8VbOAzJ1uQvQr77bVx/Ecobarter-USSD-Workflow?node-id=1-5&t=OZV8jIOUVSF64ph9-1)

---
## Project Status
This is an MVP (Minimum Viable Product) focusing on core USSD functionality. Future phases will include multi-language support and advanced analytics.