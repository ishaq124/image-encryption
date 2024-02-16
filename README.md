# image-encryption

#index js :

#!/usr/bin/env node

/**
 * imcrypt
 * An image encryption node-js cli
 *
 * 
 */

const init = require('./utils/init');
const cli = require('./utils/cli');
const encrypt = require('./utils/encrypt');
const decrypt = require('./utils/decrypt');

const input = cli.input;
const flags = cli.flags;
const { clear } = flags;

(async () => {
	init({ clear });

	input.includes(`help`) && cli.showHelp(0);
	// check if encrypt is present in flags object
	if (flags.encrypt) {
		await encrypt(flags);
	} else if (flags.decrypt) {
		await decrypt(flags);
	}

	// footer to show when the program is finished

	const chalk = (await import(`chalk`)).default;

	// print Give it a star on github: https://github.com/theninza/imcrypt with chalk and bgMagenta
	console.log(
		chalk.bgMagenta(` Give it a star on github: `) +
			chalk.bold(` https://github.com/theninza/imcrypt `)
	);
})();




#cli.js:



const meow = require('meow');
const meowHelp = require('cli-meow-help');

const flags = {
	encrypt: {
		typs: `string`,
		desc: `The image to encrypt`,
		alias: `e`
	},
	decrypt: {
		typs: `string`,
		desc: `The image to decrypt`,
		alias: `d`
	},
	outputImageFileName: {
		type: `string`,
		desc: `The output image file name`,
		alias: `i`
	},
	outputKeyFileName: {
		type: `string`,
		desc: `The output key file name`,
		alias: `p`
	},
	key: {
		type: `string`,
		desc: `The key file to use for decryption`,
		alias: `k`
	},
	clear: {
		type: `boolean`,
		default: false,
		alias: `c`,
		desc: `Clear the console`
	},
	noClear: {
		type: `boolean`,
		default: true,
		desc: `Don't clear the console`
	},
	version: {
		type: `boolean`,
		alias: `v`,
		desc: `Print CLI version`
	}
};

const commands = {
	help: { desc: `Print help info` }
};

const helpText = meowHelp({
	name: `imcrypt`,
	flags,
	commands
});

const options = {
	inferType: true,
	description: false,
	hardRejection: false,
	flags
};

const cli = meow(helpText, options);

// adding commands
cli.commands = commands;

module.exports = cli;





#decryt.js:



const alert = require('cli-alerts');
const fs = require('fs');
const jimp = require('jimp');
const path = require('path');

const decrypt = async flags => {
	// check if flags contain decrypt flag
	if (flags.encrypt) {
		alert({
			type: `warning`,
			name: `Invalid combination of flags`,
			msg: `Cannot use both --encrypt and --decrypt flags together`
		});
		process.exit(1);
	}

	// find the value of the decrypt flag
	const filePath = flags.decrypt;

	// check if the filePath is a valid file path
	if (!filePath) {
		alert({
			type: `warning`,
			name: `Invalid file path`,
			msg: `Please provide a valid file path`
		});
		process.exit(1);
	}

	// check if the filePath is a valid file path
	if (!fs.existsSync(filePath)) {
		alert({
			type: `warning`,
			name: `Invalid file path`,
			msg: `Please provide a valid file path`
		});
		process.exit(1);
	}

	// check if the key is in the flags
	if (!flags.key) {
		alert({
			type: `warning`,
			name: `Invalid key`,
			msg: `Please provide a valid key with --key/-k`
		});
		process.exit(1);
	}

	try {
		const ora = (await import('ora')).default;

		const spinner = ora(`Reading Image...`).start();

		const image = await jimp.read(filePath);

		// get the image extension using jimp
		const extension = image.getExtension();

		// get the rgba values of the image
		const rgba = image.bitmap.data;

		// get the length of the rgba array
		const length = rgba.length;

		spinner.succeed(`Image read successfully`);

		const spinner2 = ora(`Getting the key`).start();

		// get the key
		const keyPath = flags.key;

		// check if the keyPath is a valid file path
		if (!fs.existsSync(keyPath)) {
			spinner2.fail(`Invalid key path`);
			alert({
				type: `error`,
				name: `Invalid key path`,
				msg: `Please provide a valid key path with --key/-k`
			});
			process.exit(1);
		}

		// get the base64 encoded key
		const key = fs.readFileSync(keyPath, 'utf8');

		// decode the key
		const keyDecoded = Buffer.from(key, 'base64');

		const keyArray = Array.from(keyDecoded);

		// check if the key is the same length as the rgba array
		if (keyArray.length !== length) {
			spinner2.fail(`Invalid key`);
			alert({
				type: `error`,
				name: `Invalid key`,
				msg: `The key is not valid`
			});

			process.exit(1);
		}

		spinner2.succeed(`Key read successfully`);

		const spinner3 = ora(`Decrypting...`).start();

		// loop through the rgba array
		for (let i = 0; i < length; i++) {
			const k = keyArray[i];
			rgba[i] = rgba[i] ^ k;
		}

		// save the image with _decrypted appended to the file name, mimeType and the new extension
		image.bitmap.data = rgba;

		spinner3.succeed(`Decryption successful`);

		// save the image
		// get file base name before _encrypted

		const spinner4 = ora(`Saving image...`).start();

		const fileName = path
			.basename(filePath)

			// remove _encrypted from the file name if present
			.replace(/\_encrypted$/, '');

		// remove the extension from the file name
		let fileNameWithoutExtension = `${fileName.split('.')[0]}_decrypted`;

		if (flags.outputImageFileName) {
			fileNameWithoutExtension = flags.outputImageFileName.split('.')[0];
		}

		if (fs.existsSync(`${fileNameWithoutExtension}.${extension}`)) {
			console.log(flags);
			spinner4.fail(`Output image file already exists`);
			alert({
				type: `error`,
				name: `Output image file already exists: ${fileNameWithoutExtension}.${extension}`,
				msg: `Please provide a different file name with --outputImageFileName/-i flag`
			});
			process.exit(1);
		}

		image.write(`${fileNameWithoutExtension}.${extension}`);

		spinner4.succeed(`Image saved successfully`);

		alert({
			type: `success`,
			name: `Success`,
			msg: `Image decrypted successfully\n
			Decrypted Image: ${fileNameWithoutExtension}.${extension}`
		});
	} catch (err) {
		alert({
			type: `error`,
			name: `Error`,
			msg: `${err}`
		});
	}
};

module.exports = decrypt;
















