# Email Signature Generator

A modern, professional email signature generator that creates consistent email signatures with export options for common email clients.

## Live Demo
Try it out: [https://mlaplante.github.io/EmailSignature/](https://mlaplante.github.io/EmailSignature/)

## Overview

The Email Signature Generator is a web-based application that helps you create beautiful, professional email signatures with a clean and intuitive interface. It generates consistent signatures that can be easily exported and imported into various email clients including Gmail, macOS Mail, and iOS Mail.

## Features

- **Clean, Modern Interface**: Beautiful user interface with a professional design aesthetic
- **Brand Selector**: Choose from 10 professionally curated color schemes to match your brand identity
- **Live Preview**: See your email signature update in real-time as you type
- **Dark/Light Theme**: Toggle between dark and light themes for comfortable viewing
- **Multiple Fields**: Include your name, job title, company name, phone number, and an optional booking link
- **Custom Logo Upload**: Upload your own logo or use the default branded logo
- **One-Click Copy**: Copy your signature to clipboard with a single click
- **Export Guides**: Built-in instructions for importing signatures into common email clients:
  - Gmail
  - macOS Mail
  - iOS Mail
- **Responsive Design**: Works seamlessly on desktop and mobile devices
- **No Dependencies**: Standalone HTML file with no external dependencies required
- **Privacy-Focused**: All processing happens in your browser - no data is sent to any server

## Usage

### Getting Started

1. Open `index.html` in your web browser or visit the [live demo](https://mlaplante.github.io/EmailSignature/)
2. **Select your brand colors**: Choose a color scheme from the brand selector dropdown that matches your company's brand identity
3. Fill in your information in the form fields:
   - **Brand / Color Theme**: Select from 10 curated color schemes (Default Warm Terra, Professional Blue, Corporate Navy, Creative Purple, Eco Green, Energetic Orange, Modern Teal, Elegant Burgundy, Tech Cyan, Minimalist Gray)
   - **Company Name**: Optional company name to display in your signature
   - **Name**: Your full name
   - **Job Title**: Your professional title or role
   - **Phone Number**: Your contact number
   - **Book Me Link**: Optional link to your booking/scheduling page (e.g., Calendly)
   - **Custom Logo**: Upload your own logo image (or use the default branded icon)
4. Watch your signature preview update in real-time with your selected brand colors
5. Click "Copy Signature" to copy the formatted signature to your clipboard
6. Click "How to Import?" to see instructions for your email client

### Importing to Email Clients

The application includes detailed step-by-step instructions for importing your signature into:

- **Gmail**: Web-based email client instructions
- **macOS Mail**: Desktop email client for Mac users
- **iOS Mail**: Mobile email client for iPhone and iPad

Simply click the "How to Import?" button to access platform-specific instructions.

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

#### Brand Color Schemes

The generator includes 10 professionally curated brand color schemes:

1. **Default (Warm Terra)** - Warm, earthy tones with coral accent (#d97757)
2. **Professional Blue** - Classic professional blue (#2563eb)
3. **Corporate Navy** - Deep, trustworthy navy blue (#1e40af)
4. **Creative Purple** - Bold, creative purple (#9333ea)
5. **Eco Green** - Natural, eco-friendly green (#059669)
6. **Energetic Orange** - Vibrant, energetic orange (#ea580c)
7. **Modern Teal** - Contemporary teal (#0891b2)
8. **Elegant Burgundy** - Sophisticated burgundy red (#9f1239)
9. **Tech Cyan** - Modern tech cyan (#06b6d4)
10. **Minimalist Gray** - Clean, minimalist gray (#4b5563)

Each brand preset includes coordinated colors for:
- Accent color (for links and logo)
- Primary text color (for name)
- Muted text color (for title, company, and contact info)

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
