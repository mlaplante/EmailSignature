# Email Signature Generator

A modern, professional email signature generator that creates consistent email signatures with export options for common email clients.

## Live Demo
Try it out: [https://mlaplante.github.io/EmailSignature/](https://mlaplante.github.io/EmailSignature/)

## Overview

The Email Signature Generator is a web-based application that helps you create beautiful, professional email signatures with a clean and intuitive interface. It generates consistent signatures that can be easily exported and imported into various email clients including Gmail, macOS Mail, and iOS Mail.

## Features

- **Clean, Modern Interface**: Beautiful user interface with a professional design aesthetic
- **Live Preview**: See your email signature update in real-time as you type
- **Email Client Preview Modes**: Switch between Standard, Gmail, and Outlook preview modes to see how your signature will appear in different email clients
- **Dark/Light Theme**: Toggle between dark and light themes for comfortable viewing
- **Multiple Fields**: Include your name, job title, phone number, and an optional booking link
- **Custom Logo Upload**: Upload your own logo image to personalize your signature
- **One-Click Copy**: Copy your signature to clipboard with a single click
- **Export Guides**: Built-in instructions for importing signatures into common email clients:
  - Gmail
  - Outlook
  - macOS Mail
  - iOS Mail
- **Responsive Design**: Works seamlessly on desktop and mobile devices
- **No Dependencies**: Standalone HTML file with no external dependencies required
- **Privacy-Focused**: All processing happens in your browser - no data is sent to any server

## Usage

### Getting Started

1. Open `index.html` in your web browser or visit the [live demo](https://mlaplante.github.io/EmailSignature/)
2. Fill in your information in the form fields:
   - **Name**: Your full name
   - **Job Title**: Your professional title or role
   - **Phone Number**: Your contact number
   - **Book Me Link**: Optional link to your booking/scheduling page (e.g., Calendly)
   - **Custom Logo**: Optional image to personalize your signature
3. Use the preview mode tabs to see how your signature will look:
   - **Standard**: Default professional appearance
   - **Gmail**: How your signature will appear in Gmail with Arial font and Gmail-specific styling
   - **Outlook**: How your signature will appear in Outlook with Segoe UI font and Outlook-specific styling
4. Watch your signature preview update in real-time
5. Click "Copy Signature" to copy the formatted signature to your clipboard (always copies the standard version for best compatibility)
6. Click "How to Import?" to see instructions for your email client

### Importing to Email Clients

The application includes detailed step-by-step instructions for importing your signature into:

- **Gmail**: Web-based email client instructions
- **Outlook**: Desktop email client for Windows and Mac users
- **macOS Mail**: Desktop email client for Mac users
- **iOS Mail**: Mobile email client for iPhone and iPad

Simply click the "How to Import?" button to access platform-specific instructions. The copied signature works in all email clients, and the preview tabs help you understand how it will appear in different environments.

## Technical Details

### Technologies Used

- **HTML5**: Semantic markup
- **CSS3**: Modern styling with CSS variables for theming
- **Vanilla JavaScript**: No frameworks or libraries required
- **Google Fonts**: Crimson Pro and DM Sans for typography

### Browser Compatibility

Works on all modern browsers including:
- Chrome/Edge (latest)
- Firefox (latest)
- Safari (latest)
- Mobile browsers (iOS Safari, Chrome Mobile)

### Customization

The signature generator uses CSS variables for easy theming customization. You can modify the color scheme by adjusting the variables in the `:root` and `[data-theme="dark"]` selectors.

## Installation

No installation required! Simply download or clone the repository and open `index.html` in your web browser, or visit the [live demo](https://mlaplante.github.io/EmailSignature/).

```bash
git clone https://github.com/mlaplante/EmailSignature.git
cd EmailSignature
open index.html
```

Or you can host it on any web server or GitHub Pages for easy access.

## License

This project is open source and available for personal and commercial use.

## Contributing

Contributions are welcome! Feel free to submit issues or pull requests to improve the email signature generator.
