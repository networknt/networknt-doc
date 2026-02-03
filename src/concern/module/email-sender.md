# Email Sender Module

The `email-sender` module provides a simple utility to send emails using Jakarta Mail (formerly JavaMail). It is designed to be used within the Light-4j framework and integrates with the configuration system.

## Introduction

This module simplifies the process of sending emails from your application. It supports:
-   Sending both plain text and HTML emails.
-   Sending emails with attachments.
-   Template variable replacement for dynamic email bodies.
-   Configuration via `email.yml` (localized or centralized).

## Configuration

The module is configured using the `email.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `host` | SMTP server host name or IP address. | `mail.lightapi.net` |
| `port` | SMTP server port number (typical values: 25, 587, 465). | `587` |
| `user` | SMTP username or sender email address. | `noreply@lightapi.net` |
| `pass` | SMTP password. Should be encrypted or provided via enviroment variable `EMAIL_PASS`. | `password` |
| `debug` | Enable Jakarta Mail debug output. | `true` |
| `auth` | Enable SMTP authentication. | `true` |

### Example `email.yml`

```yaml
host: smtp.gmail.com
port: 587
user: your.email@gmail.com
pass: your_app_password
debug: false
auth: true
```

> **Security Note**: Never commit plain text passwords to version control. Use [Decryptor Module](decryptor.md) to encrypt sensitive values in configuration files or use environment variables.

## Usage

### dependency

Add the following dependency to your `pom.xml`.

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>email-sender</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

### Sending a Simple Email

```java
import com.networknt.email.EmailSender;

public void send() {
    EmailSender sender = new EmailSender();
    try {
        sender.sendMail("recipient@example.com", "Subject Line", "<h1>Hello World</h1>");
    } catch (MessagingException e) {
        logger.error("Failed to send email", e);
    }
}
```

### Sending an Email with Attachment

```java
public void sendWithAttachment() {
    EmailSender sender = new EmailSender();
    try {
        sender.sendMailWithAttachment(
            "recipient@example.com", 
            "Report", 
            "Please find the attached report.", 
            "/tmp/report.pdf"
        );
    } catch (MessagingException e) {
        logger.error("Failed to send email", e);
    }
}
```

### Template Replacement

The `EmailSender` provides a utility to replace variables in a template string. Variables are defined as `[key]`.

```java
Map<String, String> replacements = new HashMap<>();
replacements.put("name", "John Doe");
replacements.put("link", "https://example.com/activate");

String template = "<p>Hi [name],</p><p>Click <a href='[link]'>here</a> to activate.</p>";
String content = EmailSender.replaceTokens(template, replacements);

sender.sendMail("john@example.com", "Activation", content);
```
