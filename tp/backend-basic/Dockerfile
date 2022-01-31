# Choose a java JRE
FROM openjdk:11

# Add the compiled java (aka bytecode, aka .class)
COPY . /usr/src/basic
WORKDIR /usr/src/basic

# Run the Java with: “java Main” command.
RUN javac Main.java
CMD ["java", "Main"]
