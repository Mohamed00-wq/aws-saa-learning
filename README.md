1. Understand the requirement
        ↓
2. Design the architecture
        ↓
3. Draw the diagram
        ↓
4. Choose the AWS services
        ↓
5. Explain why each service is there
        ↓
6. Build it with AWS CLI / Console
        ↓
7. Test the architecture
        ↓
8. Update the diagram if needed



                 YOU                   
                 │
                 ▼
        Understand the goal
                 │
                 ▼
        Design architecture
                 │
                 ▼
          Draw the diagram
                 │
                 ▼
        Choose AWS services
                 │
                 ▼
          Write commands
                 │
                 ▼
          Build on AWS
                 │
                 ▼
             Test it
                 │
                 ▼
        Document what you learned





                    Internet
                       │
                       ▼
                  Route 53
                       │
                       ▼
              Application Load
                  Balancer
                /           \
               ▼             ▼
             AZ-A           AZ-B
          ┌─────────┐    ┌─────────┐
          │   EC2   │    │   EC2   │
          └────┬────┘    └────┬────┘
               │              │
               └──────┬───────┘
                      ▼
                 RDS Multi-AZ