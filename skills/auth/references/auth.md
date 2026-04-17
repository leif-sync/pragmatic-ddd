```typescript
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { magicLink, emailOTP } from "better-auth/plugins";
import { drizzleConnection, resend } from "@/shared/infrastructure/bootstrap";
import {
  GOOGLE_CLIENT_ID,
  GOOGLE_CLIENT_SECRET,
  RESEND_FROM_EMAIL,
} from "@/shared/infrastructure/load-env";
import { EmailMagicLinkTemplate } from "./templates/EmailMagicLinkTemplate";
import { EmailOTPTemplate } from "./templates/EmailOTPTemplate";
import * as schema from "@/shared/infrastructure/drizzle-postgres/schema";
import { todo } from "@/shared/domain/utils/todo";

export const auth = betterAuth({
  database: drizzleAdapter(drizzleConnection, {
    provider: "pg",
    usePlural: true,
    schema,
  }),
  socialProviders: {
    google: {
      clientId: GOOGLE_CLIENT_ID,
      clientSecret: GOOGLE_CLIENT_SECRET,
    },
  },
  plugins: [
    magicLink({
      sendMagicLink: async ({ email, url }) => {
        const { error } = await resend.emails.send({
          from: RESEND_FROM_EMAIL,
          to: email,
          subject: "Your magic link for Shortly",
          react: EmailMagicLinkTemplate({ url }),
        });

        if (error) {
          todo.warn(
            "Implement proper error handling for failed magic link email sending",
          );
          console.error("Failed to send magic link email:", error);
        }
      },
    }),
    emailOTP({
      async sendVerificationOTP({ email, otp }) {
        const { error } = await resend.emails.send({
          from: RESEND_FROM_EMAIL,
          to: email,
          subject: "Your verification code for Shortly",
          react: EmailOTPTemplate({ otp }),
        });

        if (error) {
          todo.warn(
            "Implement proper error handling for failed OTP email sending",
          );
          console.error("Failed to send OTP email:", error);
        }
      },
    }),
  ],
  advanced: {
    database: {
      generateId: "uuid",
    },
  },
});
```