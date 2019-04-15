# AndroidDynamicFlavor
## Description
A groovy build script that loads a JSON file and controls dynamically:
- Flavors
- Variants colors
- Variants images (With auto-resolution generation)
## Use cases
This could be used for Whitelabel applications, with a normal configuration and visuals, but, with some variants.
Example: An App that needs to change it's colors(Or images) when in a specific Flavor

** Every resource get's overriden on the Final APK, so every generated Flavor 'll have only it's resources **


```groovy
import groovy.json.JsonSlurper
import groovy.json.JsonBuilder
import groovy.util.XmlParser
import java.nio.file.Files
import groovy.xml.XmlUtil
import static java.awt.RenderingHints.*
import java.awt.image.BufferedImage
import javax.imageio.ImageIO

import java.util.regex.Matcher
import java.util.regex.Pattern

def log (String str) {
	project.logger.lifecycle ("[Whitelabel configuration] " + str)
}

def log (Integer integer) {
	log (integer.toString ())
}

def log (Object obj) {
	log (new JsonBuilder (obj).toPrettyString ())
}

def findMatch (pattern, source) {
	// Just create a regex matcher
	Matcher matcher = pattern.matcher (source)
	// If has at leas one match, return the first
	if (matcher.find ()) {
		return matcher
	} else {
		return null
	}
}

def getCurrentFlavor (flavorList) {
	// Get the gradle wrapper
	Gradle gradle = getGradle ()

	// Get the task requests strings
	String tskReqStr = gradle.getStartParameter ().getTaskRequests ().toString ()
	// Get the task names strings
	String tskNames = gradle.startParameter.taskNames.toString ()

	// Create a regex expression using the given flavorList
	def flavorStr = flavorList.join ("|")
	def pattern = Pattern.compile ("(?:assemble|bundle|generate)($flavorStr)(release|debug)?", Pattern.CASE_INSENSITIVE)

	// Try to run the created pattern in the task requests
	def regexMatch = findMatch (pattern, tskReqStr)
	if (regexMatch != null) return regexMatch.group (1).toLowerCase ()
	// Try to run the created pattern in the task names
	regexMatch = findMatch (pattern, tskNames)
	if (regexMatch != null) return regexMatch.group (1).toLowerCase ()
	// No match has been found
	println "NO MATCH FOUND"
	return ""
}

def handleColors (flavorDirectory, flavorConfig, configuration) {
	// Reads the original colors.xml
	def originalColorsXml = new XmlSlurper ().parse (file ("./src/main/res/values/colors.xml"))

	// Overrides the colors provided in the configuration
	originalColorsXml.color.each {
		//log ("${it.@name} -> ${it.text()}")

		//configuration.flavors.find { it.name.toLowerCase () == currentFlavor }
		flavorConfig.colors.each { colorResource ->
			log ("COLOR ${colorResource.key} -> ${colorResource.value}")

			if (it.@name == colorResource.key) {
				//log ("The property ${it.@name} has been matched and 'll be replaced with ${colorResource.value}")
				it.replaceBody colorResource.value
			} else {
				//log ("The property ${it.@name} hasn't matched")
			}
		}
	}

	// Writes to the specified thing
	def writer = new FileWriter (flavorDirectory.getAbsolutePath () + "/colors.xml")
	XmlUtil.serialize (originalColorsXml, writer)
}

def handleImages (flavorResDirectory, flavorConfig) {
	def flavorImageDirectory = new File ("$projectDir/Whitelabel/${flavorConfig.package}/images")
	if (!flavorImageDirectory.exists ()) {
		throw new GradleException ("The flavor ${flavorConfig.name} hasn't an image folder!")
	}
	// List all images inside the folder
	flavorImageDirectory.eachFile { file ->
		log ("Copying file ${file.getName ()} from flavor ${flavorConfig.name}...")

		def resolutions = [
				"ldpi"   : calculateDensityFactor (0.75),
				"mdpi"   : calculateDensityFactor (1.0),
				"hdpi"   : calculateDensityFactor (1.5),
				"xhdpi"  : calculateDensityFactor (2.0),
				"xxhdpi" : calculateDensityFactor (3.0),
				"xxxhdpi": calculateDensityFactor (4.0)
		]

		def image = ImageIO.read (file)

		resolutions.each { resolutionName, resolutionFactor ->
			Integer newWidth = image.width * resolutionFactor
			Integer newHeight = image.height * resolutionFactor

			new BufferedImage (newWidth, newHeight, image.type).with { i ->
				createGraphics ().with {
					setRenderingHint (KEY_INTERPOLATION, VALUE_INTERPOLATION_BICUBIC)
					drawImage (image, 0, 0, newWidth, newHeight, null)
					dispose ()
				}

				def destinationFile = new File (flavorResDirectory.getAbsolutePath () + "/drawable-${resolutionName}/${file.getName ()}")
				destinationFile.mkdirs ()
				ImageIO.write (i, getFileExtension (file), destinationFile)
			}
		}

	}
}

def calculateDensityFactor (density) {
	return 1 - (4 - density) / 4
}

def getFileExtension (File file) {
	def index = file.getName ().lastIndexOf (".")
	if (index != -1) {
		return file.getName ().substring (index + 1).toLowerCase ()
	} else {
		return null
	}
}

apply plugin: 'com.android.application'
android {
	compileSdkVersion 28
	defaultConfig {
		applicationId "com.example.whitelabeltest"
		minSdkVersion 21
		targetSdkVersion 28
		versionCode 1
		versionName "1.0"
		testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
	}
	buildTypes {
		release {
			minifyEnabled false
			proguardFiles getDefaultProguardFile ('proguard-android-optimize.txt'), 'proguard-rules.pro'
		}
	}

	// Loads the whitelabel configuration file
	log ("Loading .json file...")
	def configurationFile = new File ("$projectDir/Whitelabel/configuration.json")
	// Check if the file exists
	if (!configurationFile.exists ()) {
		throw new GradleException ("WhitelabelConfiguration.json file does not exists!")
	}
	// Parses the file content into an Object
	def configuration = new JsonSlurper ().parseText (configurationFile.text)
	log (configuration)

	// Generate the dynamic flavors
	log ("Generating dynamic flavors...")
	flavorDimensions "development"
	productFlavors {
		configuration.flavors.each { flavor ->
			"$flavor.name" {
				applicationIdSuffix "$flavor.package"
				buildConfigField "String", "VARIANT", "\"$flavor.name\""
			}
		}
	}

	configuration.flavors.each { flavorConfig ->
		def flavorName = flavorConfig.name

		// Copy the files
		log ("Copying resource files $flavorName...")
		def flavorDirectory = new File ("$projectDir/src/${flavorConfig.package}")

		// If the flavor directory exists, just deletes it with all the inner files
		if (flavorDirectory.exists ()) {
			log ("Flavor ${flavorName} already has a folder, deleting it...")
			flavorDirectory.deleteDir ()
		}

		// Create the flavor directory
		if (flavorDirectory.mkdirs ()) {
			log ("Flavor ${flavorName}'s root folder has been created")
		} else {
			throw new GradleException ("Error while creating directory!")
		}

		// Create the resource folder
		def flavorResDirectory = new File (flavorDirectory.getAbsolutePath () + "/res")
		if (flavorResDirectory.mkdirs ()) {
			log ("Flavor ${flavorName}'s resources folder has been created")
		} else {
			throw new GradleException ("Error while creating directory!")
		}

		// Create the resource values
		def flavorResValuesDirectory = new File (flavorResDirectory.getAbsolutePath () + "/values")
		if (flavorResValuesDirectory.mkdirs ()) {
			log ("Flavor ${flavorName}'s resources/values folder has been created")
		} else {
			throw new GradleException ("Error while creating directory!")
		}

		// Override the colors.xml file
		handleColors (flavorResValuesDirectory, flavorConfig, configuration)

		// Overrides the images
		handleImages (flavorResDirectory, flavorConfig)
	}

}

dependencies {
	implementation fileTree (dir: 'libs', include: ['*.jar'])
	implementation 'com.android.support:appcompat-v7:28.0.0'
	implementation 'com.android.support.constraint:constraint-layout:1.1.3'
	testImplementation 'junit:junit:4.12'
	androidTestImplementation 'com.android.support.test:runner:1.0.2'
	androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}

```
